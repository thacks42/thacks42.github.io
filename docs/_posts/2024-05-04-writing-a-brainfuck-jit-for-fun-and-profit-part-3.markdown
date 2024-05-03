---
layout: post
title:  "Writing a Brainfuck JIT for fun and profit. Part 3"
---


## Rewriting in C++

The first decision we have to make is about our intermediate representation: We need something which is both easy to generate from our brainfuck input,
as well as being straight forward to manipulate in our optimization passes. It should, if at all possible, give us a little bit of structure, which we can use as a guide for our optimization algorithms.

As previously alluded I have no previous experience in compiler development so all decisions I made were a mix of "I *think* I heard that compiler people do something like this"
and just making up stuff as I went on. So if anyone reading this happens to actually know how compilers are done *properly*, please don't be too hard on me :)

With the disclaimer out of the way, let's take a look at how our intermediate representation might look like. I decided to use structs to represent different "nodes" which perform operations,
such that the type of node specifies the operation and its content provides the operands.

```cpp
struct add_ptr{ //add a value to the pointer
    int amount;
};

struct add_cell{ //add a value to the current cell
    int amount;
};

struct set_cell{ //set a cell at "[pointer + offset]" to "value" 
    int offset;
    int value;
};

struct cell_arithmetic{ //add "value" at "[source_offset]" times "factor" to "[target_offset]"
    int source_offset;
    int target_offset;
    int factor;
};

...

using bf_node = std::variant<add_ptr, add_cell, set_cell, cell_arithmetic, ...>; //variant type to hold any of the possible nodes
```

This means that we translate both "pointer increment" and "pointer decrement" to the same intermediate representation node just using a different `amount` constant (1 for increment, -1 for decrement).
The same is true for the "cell increment" and "cell decrement"

## About trees and forests

Since brainfuck has a relatively simple branching structure (there are no function calls, no gotos, no if/then/else constructs etc. just while loops),
we can represent our whole program as an array of nodes with special "loop" nodes, which contain an array of nodes representing the content of the loop:

```cpp
struct repeat{ //repeats the "inner" section as long as the current cell is non-zero
    std::vector<bf_node> inner;
};
```

Then we can revisit our simple parser from part 1 and modify it to generate a parse tree for us:

```cpp
bf_tree bf_to_tree(std::string_view input){
    std::vector<bf_node> result;                // the array that holds all of our nodes
    std::stack<std::vector<bf_node>*> current;  // keep track of the array we are currently adding nodes to
    current.push(&result);                      // start by writing to the outer-most array
    
    for(size_t i = 0; i < input.size(); i++){
        switch(input[i]){
            case '>':
                current.top()->push_back(add_ptr(1));
                break;
            case '<':
                current.top()->push_back(add_ptr(-1));
                break;
            case '+':
                current.top()->push_back(add_cell(1));
                break;
            case '-':
                current.top()->push_back(add_cell(-1));
                break;
            case '[':
            {
                current.top()->push_back(repeat{}); //create a "repeat" node
                
                auto& new_top = std::get<repeat>(current.top()->back()).inner;
                current.push(&new_top); //from now on add all new nodes into the array within the "repeat" node
                break;
            }
            case ']':
                if(current.size() == 1){
                    return {};
                }
                current.pop(); //step out of the array within the loop node and continue adding nodes to the array that contains it
                break;
            default:
                break;
            }
            ...
        }
    }
    return result;
}
```

And also write a small function to "pretty print" our parse tree for us to examine:

```cpp
void spaces(int n){ //print `n` spaces
    for(int i = 0; i < n; i++) std::cout << ' ';
}

void print(const std::vector<bf_node>& bf_nodes, int indent = 0){
    for(size_t i = 0; i < bf_nodes.size(); i++){
        auto& v = bf_nodes[i];  //get the current node
        std::visit(overloaded{  //print the appropriate content
            [indent](add_ptr x)         { spaces(indent); std::cout << "add ptr " << x.amount << "\n"; },
            [indent](add_cell x)        { spaces(indent); std::cout << "add cell " << x.amount << "\n"; },
            [indent](set_cell x)        { spaces(indent); std::cout << "set cell " << x.offset << " to " << x.value << "\n"; },
            [indent](cell_arithmetic x) { spaces(indent); std::cout << "add cells   source: " << x.source_offset << " target: " << x.target_offset << " factor: " << x.factor << "\n"; },
            [indent](repeat x)          { spaces(indent); std::cout << "(repeat)[\n"; print(x.inner, indent + 4); spaces(indent); std::cout << "]\n";},
            [indent](output_char x)     { spaces(indent); std::cout << "output\n"; },
            ...
        },v);
    }
}
```

So let's combine the two and look at some intermediate representation! In part 1 we looked at a program to print "!", so what does that look like after parsing?

```
print "!"

+++++++++++++++++++++++++++++++++       increment the current cell 33 times
.                                       and output its value: ASCII 33 is an exclamation mark so this will print "!"
```

turns into:

```
add cell 1
add cell 1
add cell 1
... {repeats 33 times overall}
add cell 1
output
```

What about the program to add the content of two cells?
```
[           if the current cell is not zero:
    -       decrement the current cell
    >       move to the next cell
    +       increment the cell
    <       move back to the first cell
]           repeat until the current cell is zero
```

turns into:
```
(repeat)[
    add cell -1
    add ptr 1
    add cell 1
    add ptr -1
]
```

## Writing the first optimization-pass

This seems to work well enough, which means that we can finally start writing optimizations. So let us start by folding multiple consecutive "add cell" nodes into a single one:
```cpp
void fold_arithmetic(std::vector<bf_node>& tree){
    for(size_t i = 0; i < tree.size(); i++){                // iterate over all nodes in the tree
        bf_node& node = tree[i];                            
        if(add_cell* op = std::get_if<add_cell>(&node)){    // check if the current node is of the "add_cell" type
            int amount = op->amount;                        // get the amount added to the cell
            size_t j = i+1;
            for(; j < tree.size(); j++){                    // find all nodes of the same type after this one
                if(add_cell* op_2 =
                    std::get_if<add_cell>(&tree[j])){       // if the next node is also an "add_cell"...
                    amount += op_2->amount;                 // ...add up the amounts
                }
                else{
                    break;                                  // stop iterating if an instruction of a different type shows up
                }
            }
            op->amount = amount;                            // replace the amount in the first "add_cell" node by the total...
            tree.erase(tree.begin()+i+1, tree.begin() + j); // ...and remove all the other "add_cell" nodes that came after
        }
        else if(repeat* l = std::get_if<repeat>(&node)){    // if we encounter a loop node...
            fold_arithmetic(l->inner);                      // recursively apply this optimization to all nodes in the loop
        }
    }
}
```

Applying this optimization to the "print !" example yields what we'd expect:

```
add cell 33
output
```

But we already got this far in our previous compiler, so how can we leverage our intermediate representation in a meaningful way?
First of all we should take note of the fact that our `fold_arithmetic` already handles cell additions as well as cell subtractions due to them both being parsed into `add_cell` nodes,
so our intermediate representation already paid off. 

Secondly this also handles the (much rarer) case of "bad code" where a cell increment is followed directly by a cell decrement and the other way around,
as well as any arbitrary mixed sequence of the two, which our previous compiler could not deal with.

Thirdly we can turn this function into a template which allows us to use the same function to also handle `add_ptr` nodes in the same way without any code duplication.

And if we add a function to detect the "set cell to zero" pattern, then we've successfully outclassed the previous compiler. But none of this seems really groundbreaking, 
what we're really looking for is to go beyond what was meaningfully possible, so how about a single optimization function that takes care of all "cell to cell" additions and subtractions?
All those patterns we previously identified as "conceptually similar" but weren't able to properly handle due to the limitations of a single pass compiler...

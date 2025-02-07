# Path-Expression-Traversal-System

![The logo, depicting a cat slowly becoming a mesh network](./readme_images/PETS.png)

PETS, a system to store linked distributed data with traversal functions

## Architecture

```mermaid
sequenceDiagram
    participant API
    participant cent as Central server

    note over API: where request start


    note over cent: A server which listens to http request<br/>and answer/forward queries.<br/>All servers are of this type, but their<br/>databases very<br/>and the central database is quite<br/>different with only one node*.

    API->>+cent: Pick/crafted_by*
    note left of cent: sent over http api

    loop until not stored
        create participant DB_A as DB
        cent->>DB_A: 
        note over DB_A: This database is just a node<br/>which edges point to items in other servers
        destroy DB_A
        DB_A->>cent: 
    end
    note left of DB_A: won't loop since only one node



    participant Tool_company as The tool company
    cent->>+Tool_company: Pick/crafted_by*


    loop until not stored
        create participant DB_B as DB
        Tool_company->> DB_B: 
        note over DB_B: A graph data base<br/>keeps track of nodes<br/>and relations between them.<br/>Also have node to denote<br/>when to pass the query to<br/>different server.
        destroy DB_B
        DB_B->>Tool_company: 
    end

    note left of DB_B: Pickaxe don't have<br/>nodes stone or stick<br/>returns server contact<br/>information.

    
    note right of Tool_company: outgoing querys can be sent in parallel
    participant Mason_LTD as Masons LTD
    Tool_company->>+Mason_LTD: Stone/crafted_by*


    loop until not stored
        create participant DB_C as DB
        Mason_LTD->>DB_C: 
        destroy DB_C 
        DB_C->>Mason_LTD: 
    end
    note left of DB_C: Stone don't have crafted by<br/>and is att end of query, return stone


    Mason_LTD->>-Tool_company: Data of Stone

    participant Wood_INC as Wood INC

    Tool_company->>+Wood_INC: Stick/crafted_by*
    loop until not stored
        create participant DB_D as DB
        Wood_INC->>DB_D: 
        destroy DB_D
        DB_D->>Wood_INC: 
    end
    note left of DB_D: This query can be resolved<br/>without traversing to other<br/>servers<br/>Stick -> Plank -> Log

    Wood_INC->>-Tool_company: Data of Log
    Tool_company->>-cent: [Data of Stone, Data of Log]
    cent->>-API: [Data of Stone, Data of Log]

```

## Query structure

The query structure was designed for simplicity and not fines, the goal was an easy way to write path expressions with loops

```mermaid
graph LR;
    S -->|Pickaxe| Pickaxe;
    S -->|Stick| Stick;

    Pickaxe -->|foundAt| Mineshaft
    Pickaxe -->|obtainedBy| Pickaxe_From_Stick_And_Stone_Recipe;
    Pickaxe_From_Stick_And_Stone_Recipe -->|hasInput| Stick;
    Pickaxe_From_Stick_And_Stone_Recipe -->|hasInput| Cobblestone;

    Mineshaft -->|rarity| Rare
    Pickaxe_From_Stick_And_Stone_Recipe -->|rarity| Common;

    Stick -->|obtainedBy| Stick_From_Planks_Recipe;
    Stick_From_Planks_Recipe -->|hasInput| Plank;

    Plank -->|obtainedBy| Plank_From_Logs_Recipe;
    Plank_From_Logs_Recipe -->|hasInput| Log
```

### Example 1, Simple traversal

To follow a simple path, first have the starting node (s in this case a we have not implemented a dht to resolve node location) followed by the edges name separated by `/`

`S/Pickaxe/obtainedBy/crafting_recipe/hasInput`

The example will start att pickaxe and follow edge `obtainedBy` to `Pickaxe_From_Stick_And_Stone_Recipe`
where the query will split and go to both `Cobblestone` and `stick`.
Since this is the end of the query they are returned

### Example 2, Loop

Looping expressions, matching more than once, allowing for following a path of unknown length. The syntax is the to add a star around a group ``{...}*``

``S/Pickaxe/{obtainedBy/hasInput}*``

Will see what pick is made of recursively down to its minimal component, the paths are

```text
Pickaxe --> Pickaxe_From_Stick_And_Stone_Recipe --> Stick --> Stick_From_Planks_Recipe --> Plank --> Plank_From_Logs_Recipe --> Log
Pickaxe --> Pickaxe_From_Stick_And_Stone_Recipe --> Cobblestone
```

both Cobblestone and Log would be returned

### Example 3, Or

allows a path traversal to follow either edge

``S/Pickaxe/{obtainedBy/rarity|foundAt}/rarity``

```text
Pickaxe --> Pickaxe_From_Stick_And_Stone_Recipe --> Common
Pickaxe --> Mineshaft --> Rare
```

### Example 4, AND

Only allows the query to continue if both edges exist on the node, both are traversed

``S/Pickaxe/{obtainedBy & foundAt}/rarity`` would return

```text
Pickaxe --> Pickaxe_From_Stick_And_Stone_Recipe --> Common
Pickaxe --> Mineshaft --> Rare
```

``S/Stick/{obtainedBy & foundAt}/rarity`` would return nothing as stick dont have the edge foundAt

### Example 5, groups {}

TODO EXPLAIN MORE

S/Pick/{(made_of & Crafting_recipie)/made_of}

### example arguments (), TO BE DECIDED

arguments could be added to loop operator?

## Example of internal structure of a query

Lets take an example query af show its internal evaluation

``S/Pickaxe/{obtainedBy/hasInput}*``

This is then converted to a tree structure of operations, where the leafs are edges and

<!-- Note to readers, this look incredibly like the state machines that regex compiles to -->
```mermaid
graph TD;
    r([root]);
    r -->|left| s
    r -->|right| 2

    2([/]);
    2 -->|left| Pickaxe
    2 -->|right| 3

    3([\*]);
    3 -->|left| 4
    3 -->|right| NULL

    4([/])
    4 -->|left| obtainedBy
    4 -->|right| hasInput
```

### An example of evaluation

lets say that we are on edge ``obtainedBy``, and we want to know whats next.
By looking at the parent we know that we are on the left side of an *traverse*
and the next edge is the one on the right of the traverse, ``hasInput``

if whe should get the next node from ``hasInput`` we can again look att the parent
and se that we are on the right side of the *traverse*,
to find the next node we need to look higher, the *traverse*'s parent.
This gives us the knowledge that we are on the left side of *loop* operator (aka *zero or more*)
We then have two possible options continue right or redo the left side.
by evaluating the left side we get ``obtainedBy`` again, showing us that the *loop* works.
the right sides gives us NULL, the end of the query an valid position to return

## go style pseudo code

Note that this pseudo code

```go
type TraverseNode Struct{
    Parent  *Node
    Left    *Node
    Right   *Node
}

// when calling this function we need to know where this was called from, was it our parent, left or right, there for passing a pointer to caller is necessary
// the function return array of pointers to the leafs/query edges, which can be used to determine the next node(s)
func (self TraverseNode) nextEdge(caller *Node) []*LeafNode {
    // if the caller is parent, we should deced into the left branch,
    if caller == self.parent {
        return self.left.nextEdge(&self)
    }
    // when the left branch has evaluated it will call us again
    // an we than have to evaluate the right branch
    else if caller == self.left {
        self.right.nextEdge(&self)
    }
    // when the right brach has evaluated it will call us again
    // we then know we have been fully evaluated and can call our parent saying we are done
    else if caller == self.right {
        self.parent.nextEdege(&self)
    } else {
        log.fatal("i dont know what should happen here?")
    }
}

type LeafNode Struct {
    Parent      *Node
    edgeName    string
}

// we are asked what the next edge is, this LeafNode represent that edge
func (self LeafNode) nextEdge(caller *Node) []*LeafNode {
    if caller == self.parent {
        return [&self]
    } else {
        log.fatal("i dont know what should happen here?")
    }
}

type LoopNode Struct{
    Parent  *Node
    Left    *Node
    Right   *Node
}

func (self LoopNode) nextEdge(caller *Node) []*LeafNode {
    // if the caller is parent, the possible outcomes are that we match zero of the edges and move on with the right branch
    // or that we match whatever in the left brach, therefore we return the next edges
    // an therefore return 
    // if this was match one and more instead of zero or more, calling right would not be the right option as it then would progress forward without having matched anything on the left
    // maybe add an + operator which is match one or more?
    if caller == self.parent {
        return [self.left.nextEdge(&self),self.right.nextEdge(&self)]
    }
    // when the left branch has evaluated it will call us again
    // we can continue the loop so left is an valid option, but we could also exit
    // this leads to the same output as the caller was the parent
    else if caller == self.left {
        return [self.left.nextEdge(&self),self.right.nextEdge(&self)]
    }
    // when the right brach has evaluated it will call us again
    // we then know we have been fully evaluated and can call our parent saying we are done
    else if caller == self.right {
        self.parent.nextEdge(&self)
    } else {
        log.fatal("i dont know what should happen here?")
    }
}

type OrNode Struct{
    Parent  *Node
    Left    *Node
    Right   *Node
}

func (self OrNode) nextEdge(caller *Node) []*LeafNode {
    // if the parent calls us we could either match each side, so both sides are an alternative
    if caller == self.parent {
        return [self.left.nextEdge(&self),self.right.nextEdge(&self)]
    }
    // if the left side calls us we have then completed one of the options and are fully evaluated, an the call our parent saying we are done, and let them get the next edge
    else if caller == self.left {
        self.parent.nextEdge(&self)
    }
    // same ass above
    else if caller == self.right {
        self.parent.nextEdge(&self)
    } else {
        log.fatal("i dont know what should happen here?")
    }
}

```

## Parsing ontologies to GoLang

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

// the parse function takes a Go map (a dictionary with key-value pairs) with a key being the node name and the value being the DataNode struct
func parse(nodeLst map[string]DataNode) map[string]DataNode { 

    // ontology is read from the servers own docker volume (server storage)
	file, err := os.Open("./shared_volume/data.ttl") 
	if err != nil {
		fmt.Println(err)
		return nodeLst
	}
	defer file.Close()

    // nodeLst map and other temp variables are declared here. the temp variables are used for storing and writing nodes/node information
	nodeLst = make(map[string]DataNode)
	var tempN DataNode  
	var tempTuple DataEdge
	var firstWord string

	scanner := bufio.NewScanner(file) 
	for scanner.Scan() {
        
        // many if statement to check the prefix of the lines in order to see type of node
		line := scanner.Text()
		if strings.HasPrefix(line, "@prefix") { // SKIP LINES STARTING WITH "@PREFIX"
			continue
		}

        // first condition to check if it could be a node
		if strings.HasPrefix(line, "minecraft:") { 
			temp := strings.TrimPrefix(line, "minecraft:")

			wrd := getWrd(temp) // FIRST WORD IN LINE
			firstWord = wrd // store in tempVar
			
        // If object has id, we know it is an actual node and save it
		} else if strings.HasPrefix(line, "	nodeOntology:hasID ") { // CHECK ID
			temp := strings.TrimPrefix(line, "	nodeOntology:hasID ")
			nodeLst[firstWord] = tempN
			wrd := getWrd(temp) // FIRST WORD IN LINE
			
			if entry, ok := nodeLst[wrd]; ok {
				entry.Edges = append(entry.Edges, tempTuple)
				nodeLst[wrd] = entry
			} // APPEND KEY NODE TO MAP OF NODES

			// CHECK FOR EDGES IN FOLLOWING ELSE IF STATEMENT
		} else if strings.HasPrefix(line, "    minecraft:obtainedBy") || (strings.HasPrefix(line, "    minecraft:hasInput")) || (strings.HasPrefix(line, "    minecraft:hasOutput") || (strings.HasPrefix(line, "    minecraft:usedInStation"))) {

			if strings.HasPrefix(line, "    minecraft:obtainedBy") {
				temp := strings.TrimPrefix(line, "    minecraft:obtainedBy minecraft:")
				wrd := getWrd(temp)
				tempTuple.EdgeName = "obtainedBy"
				tempTuple.TargetName = wrd
			} else if strings.HasPrefix(line, "    minecraft:hasInput") {
				temp := strings.TrimPrefix(line, "    minecraft:hasInput minecraft:")
				wrd := getWrd(temp)
				tempTuple.EdgeName = "hasInput"
				tempTuple.TargetName = wrd
			} else if strings.HasPrefix(line, "    minecraft:hasOutput") {
				temp := strings.TrimPrefix(line, "    minecraft:hasOutput minecraft:")
				wrd := getWrd(temp)
				tempTuple.EdgeName = "hasOutput"
				tempTuple.TargetName = wrd
			} else if strings.HasPrefix(line, "    minecraft:usedInStation") {
				temp := strings.TrimPrefix(line, "    minecraft:usedInStation minecraft:")
				wrd := getWrd(temp)
				tempTuple.EdgeName = "usedInStation"
				tempTuple.TargetName = wrd
			}
			
			if entry, ok := nodeLst[firstWord]; ok {
				entry.Edges = append(entry.Edges, tempTuple)
				nodeLst[firstWord] = entry
			} // appends edges to DataEdge[] in node "firstword"
		}
		if strings.HasSuffix(line, ";") {
			continue // NEXT LINE IN SAME NODE


		} else if strings.HasSuffix(line, ".") { // describes end of node, so reset the tempVariable firstWord
			firstWord = "" // EMPTY NODE (NEW NODE)
		} else {
			continue // NEWLINE/EMPTY SPACE
		}
	}
	if err := scanner.Err(); err != nil {
		fmt.Println("Error reading file:", err)
	}
	fmt.Println(nodeLst)
	return nodeLst
}

// getWrd gets the first word separated by a space
func getWrd(w string) string { 
	wrd := ""
	for i := range w {
		if w[i] == ' ' {
			wrd = w[0:i]
			break
		}
	}
	return wrd
}
```
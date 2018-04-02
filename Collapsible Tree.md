Collapsible Tree
================

### Introduction
The Tree layout is representative of D3's family of hierarchical layouts. It's designed to produce a 'node-link' diagram that shows the parent-child relationships between nodes.
So an advantage of this type of diagram is that it can clearly display the hierarchical relationships in the data. And the collapsible Tree allows people to interactively explore the tree instead of showing all the nodes at the same time.  
This guide is about how to construct a simple collapsible tree using d3.js 4.0.  
A big difference in d3.js 3.0 and d3.js 4.0 is the d3.svg.diagonal() function. In v3, this function can be used to draw the links between the nodes. But in v4, we have to draw the links by ourself.
  
### Setting up 
Let's start with setting up the styling for the circle that represents the nodes, the text alongside them and the links between them. Also, we load the d3 library.
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Collapsible Tree</title>
    <style>

  .node circle {
    fill: #fff;
    stroke: steelblue;
    stroke-width: 3px;
  }

  .node text { font: 12px sans-serif; }

  .link {
    fill: none;
    stroke: #ccc;
    stroke-width: 2px;
  }
    </style>
  </head>

  <body>

  <!-- load the d3.js library --> 
  <script src="http://d3js.org/d3.v3.min.js"></script>
 
  </body>
</html>
```
  
### Adding initial data
Then we add the data. Note that the data should be encoded hierarchically in JSON.
```js
var treeData = {
  name: "Top Level",
  children: [
    {
      name: "Level 2: A",
      children: [{ name: "Son of A" }, { name: "Daughter of A" }]
    },
    { name: "Level 2: B" }
  ]
};
```
  
### Setting the diagram features and adding a tree layout
Next we set up the size and margin of svg, as well as some other variables that we'll use later.  
Then we create a tree using d3.tree function and set it's size.
```js
// set margin
var margin = { top: 20, right: 90, bottom: 30, left: 90 },
  width = 960 - margin.left - margin.right,
  height = 500 - margin.top - margin.bottom;

// add svg
var svg = d3
  .select("body")
  .append("svg")
  .attr("width", width + margin.right + margin.left)
  .attr("height", height + margin.top + margin.bottom)
  // adding group into svg
  .append("g")
  // put group on the top-left
  .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

// add other variables
var i = 0,
  duration = 750,
  root;

// declare a tree 
var treemap = d3.tree().size([height, width]);
```
  
### Adding links between nodes
We first define a function to add links(Bézier curve) between parents and children.
```js
function diagonal(s, d) {
  path = `M ${s.y} ${s.x}
    C ${(s.y + d.y) / 2} ${s.x},
      ${(s.y + d.y) / 2} ${d.x},
      ${d.y} ${d.x}`;
  return path;
}
```
  
### Collapsing the tree
Next we define a function collaspe, which is used for collapsing the tree.   
If there are children for node d, then store it's children in d._children and set d.children to null, so the children of node d will be collapsed.  
```js
function collapse(d) {
  if (d.children) {
    d._children = d.children;
    d._children.forEach(collapse);
    d.children = null;
  }
}
```
  
### Click event
Then we define a function that allows us to expanding or collapse a tree by clicking nodes.  
If there are information stored in children for the clicked node, then store it's children in _children and set children to null. So after update this node's chilren will be collapsed.  
If the children is null for the clicked node, it mean that there must be no descendants of the node showing in the diagram now and after click, we should show its children(if it has). So we return the information in _children to children, and set _children to null which indicates the expanding state of the node.
```js
function click(d) {
  if (d.children) {
    d._children = d.children;
    d.children = null;
  } else {
    d.children = d._children;
    d._children = null;
  }
  update(d);
}
```
  
### Updating the tree 
In the above function we used update, which we haven't defined. Now let's write a function for updating the tree layout, so that we can interact with the diagram.  
The update function mainly consists of two part: updating the nodes and updating the links between nodes.  
This enables the diagram to show that any parent node will collapse the portion of the tree below it when clicked on. And conversly, any parent node can be clicked on again to regrow.
The code is a little long but it's not difficult to understand and I've commented for each step. 
```js
function update(source) {

  // get the positions of nodes
  var treeData = treemap(root);
  
  // compute new tree layout
  var nodes = treeData.descendants(),
  links = treeData.descendants().slice(1);
  
  // determine the horizontal spacing of the nodes
  nodes.forEach(function(d) {
  d.y = d.depth * 180;
  });
  
  
  // Dealing with nodes
  // add ids to nodes so we can select nodes by indexing
  var node = svg.selectAll("g.node").data(nodes, function(d) {
    return d.id || (d.id = ++i);
  });
  
  // enter any new nodes at the parent's previous position.
  var nodeEnter = node
    .enter()
    .append("g")
    .attr("class", "node")
    // default position is the current parent's position
    .attr("transform", function(d) {
      return "translate(" + source.y0 + "," + source.x0 + ")";
    })
    // add click event to each new node
    .on("click", click);
  // add cycle to each new node
  nodeEnter
    .append("circle")
    .attr("class", "node")
    .attr("r", 1e-6)
    // if a node has children and is collasped, then fill it with lightsteelblue
    .style("fill", function(d) {
      return d._children ? "lightsteelblue" : "#fff";
    });
  // add text to each new node
  nodeEnter
    .append("text")
    .attr("dy", ".35em")
    .attr("x", function(d) {
      return d.children || d._children ? -13 : 13;
    })
    .attr("text-anchor", function(d) {
      return d.children || d._children ? "end" : "start";
    })
    .text(function(d) {
      return d.data.name;
    });
  
  // transition entering nodes to their new position.
  var nodeUpdate = nodeEnter.merge(node);
  nodeUpdate
    .transition()
    .duration(duration)
    .attr("transform", function(d) {
      return "translate(" + d.y + "," + d.x + ")";
    });
  
  // set the styling of the entering nodes
  nodeUpdate
    .select("circle.node")
    .attr("r", 10)
    .style("fill", function(d) {
      return d._children ? "lightsteelblue" : "#fff";
    })
    .attr("cursor", "pointer");
    
  // transition exiting nodes to the parent's new position.
  var nodeExit = node
    .exit()
    .transition()
    .duration(duration)
    .attr("transform", function(d) {
      return "translate(" + source.y + "," + source.x + ")";
    })
    .remove();
  
  // "remove" the shape and text of the exiting nodes
  nodeExit.select("circle").attr("r", 1e-6);
  nodeExit.select("text").style("fill-opacity", 1e-6);
  
  
  // Dealing with links
  // select all the links
  var link = svg.selectAll("path.link").data(links, function(d) {
    return d.id;
  });
  
  // enter any new links at the parent's previous position.
  var linkEnter = link
    .enter()
    .insert("path", "g")
    .attr("class", "link")
    .attr("d", function(d) {
      var o = { x: source.x0, y: source.y0 };
      return diagonal(o, o);
    });
  
  // transition entering links to their new position.
  linkEnter.merge(link)
    .transition()
    .duration(duration)
    .attr("d", function(d) {
      return diagonal(d, d.parent);
    });
  
  // transition exiting links to the parent's new position.
  link.exit()
    .transition()
    .duration(duration)
    .attr("d", function(d) {
    var o = { x: source.x, y: source.y };
    return diagonal(o, o);
    })
    .remove();
  	
  // stash the old positions for transition.
  nodes.forEach(function(d) {
    d.x0 = d.x;
    d.y0 = d.y;
  });
}
```
  
### Constructing the tree
Finally, we can contruct the tree.  
We first compute the parent, children, height, depth for each node. 
In the second line, we set the position for the first node to be at the middle left of the screen. And then we set initial tree to show only two layer. 
Lastly, we just need to update the collapsed tree. 
```js
root = d3.hierarchy(treeData, function(d) {return d.children;});

root.x0 = height / 2;
root.y0 = 0;

// collapse nodes after the second layer
root.children.forEach(collapse);

update(root);
```
  
### Conclusion
Tree diagrams are usually crowded and I hope this tutorial can be helpful when you 
want to draw a interactive tree using D3. In this tutorial I just introduced a based structure of an collaspible tree. You can try to add more functions to it, such as interactively changing the color or direction, to make it more interesting. The first link below might be useful if you want to dwell deeper. 
  
### References
http://www.d3noob.org/2014/01/tree-diagrams-in-d3js_11.html  
https://blog.pixelingene.com/2011/07/building-a-tree-diagram-in-d3-js/  
http://bl.ocks.org/d3noob/8375092  
https://bl.ocks.org/d3noob/43a860bc0024792f8803bba8ca0d5ecd  

---
title:  "Terraform - Graph Algorithms [Go]"
layout: default
last_modified_date: 2021-07-28T08:36:00+0300
nav_order: 3

status: PUBLISHED
language: Go
short-title: Graph Algorithms
project:
    name: Terraform
    key: terraform
    home-page: https://github.com/hashicorp/terraform
tags: [graph, algorithm]
---

{% include article-meta.html article=page %}

## Context

Terraform is an open-source [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code) software tool created by HashiCorp. Terraform users define their infrastructure using declarative configuration files and apply changes to hundreds of cloud providers.

## Problem

Infrastructure resources and their dependencies form a graph. Terraform needs to implement various graph algorithms to process them. For instance, it needs to validate that there are no circular dependencies between resources, i.e. that the graph is a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) (Directed Acyclic Graph).

## Overview

Terrafrom implements a number of graph algorithms:
* [Depth-first search (DFS)](https://en.wikipedia.org/wiki/Depth-first_search).
* [Tarjan's strongly connected components algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm).
* [Transitive reduction](https://en.wikipedia.org/wiki/Transitive_reduction).
* Parallel graph walk.

## Implementation details

[Graph structure](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph.go#L9-L15). A graph is represented as a set of edges, a set of vertices, a map of outgoing edges (`downEdges`) and a map of incoming edges (`upEdges`).
```go
// Graph is used to represent a dependency graph.
type Graph struct {
	vertices  Set
	edges     Set
	downEdges map[interface{}]Set
	upEdges   map[interface{}]Set
}
```

[Connecting two vertices](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph.go#L196-L231):
```go
// Connect adds an edge with the given source and target. This is safe to
// call multiple times with the same value. Note that the same value is
// verified through pointer equality of the vertices, not through the
// value of the edge itself.
func (g *Graph) Connect(edge Edge) {
	g.init()

	source := edge.Source()
	target := edge.Target()
	sourceCode := hashcode(source)
	targetCode := hashcode(target)

	// Do we have this already? If so, don't add it again.
	if s, ok := g.downEdges[sourceCode]; ok && s.Include(target) {
		return
	}

	// Add the edge to the set
	g.edges.Add(edge)

	// Add the down edge
	s, ok := g.downEdges[sourceCode]
	if !ok {
		s = make(Set)
		g.downEdges[sourceCode] = s
	}
	s.Add(target)

	// Add the up edge
	s, ok = g.upEdges[targetCode]
	if !ok {
		s = make(Set)
		g.upEdges[targetCode] = s
	}
	s.Add(source)
}
```

[Finding the root node in a DAG](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L61-L82):
```go
// Root returns the root of the DAG, or an error.
//
// Complexity: O(V)
func (g *AcyclicGraph) Root() (Vertex, error) {
	roots := make([]Vertex, 0, 1)
	for _, v := range g.Vertices() {
		if g.upEdgesNoCopy(v).Len() == 0 {
			roots = append(roots, v)
		}
	}

	if len(roots) > 1 {
		// TODO(mitchellh): make this error message a lot better
		return nil, fmt.Errorf("multiple roots: %#v", roots)
	}

	if len(roots) == 0 {
		return nil, fmt.Errorf("no roots found")
	}

	return roots[0], nil
}
```

[Transitive reduction](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L84-L114):

```go
// TransitiveReduction performs the transitive reduction of graph g in place.
// The transitive reduction of a graph is a graph with as few edges as
// possible with the same reachability as the original graph. This means
// that if there are three nodes A => B => C, and A connects to both
// B and C, and B connects to C, then the transitive reduction is the
// same graph with only a single edge between A and B, and a single edge
// between B and C.
//
// The graph must be valid for this operation to behave properly. If
// Validate() returns an error, the behavior is undefined and the results
// will likely be unexpected.
//
// Complexity: O(V(V+E)), or asymptotically O(VE)
func (g *AcyclicGraph) TransitiveReduction() {
	// For each vertex u in graph g, do a DFS starting from each vertex
	// v such that the edge (u,v) exists (v is a direct descendant of u).
	//
	// For each v-prime reachable from v, remove the edge (u, v-prime).
	for _, u := range g.Vertices() {
		uTargets := g.downEdgesNoCopy(u)

		g.DepthFirstWalk(g.downEdgesNoCopy(u), func(v Vertex, d int) error {
			shared := uTargets.Intersection(g.downEdgesNoCopy(v))
			for _, vPrime := range shared {
				g.RemoveEdge(BasicEdge(u, vPrime))
			}

			return nil
		})
	}
}
```

[DFS](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L182-L219):
```go
// DepthFirstWalk does a depth-first walk of the graph starting from
// the vertices in start.
func (g *AcyclicGraph) DepthFirstWalk(start Set, f DepthWalkFunc) error {
	seen := make(map[Vertex]struct{})
	frontier := make([]*vertexAtDepth, 0, len(start))
	for _, v := range start {
		frontier = append(frontier, &vertexAtDepth{
			Vertex: v,
			Depth:  0,
		})
	}
	for len(frontier) > 0 {
		// Pop the current vertex
		n := len(frontier)
		current := frontier[n-1]
		frontier = frontier[:n-1]

		// Check if we've seen this already and return...
		if _, ok := seen[current.Vertex]; ok {
			continue
		}
		seen[current.Vertex] = struct{}{}

		// Visit the current node
		if err := f(current.Vertex, current.Depth); err != nil {
			return err
		}

		for _, v := range g.downEdgesNoCopy(current.Vertex) {
			frontier = append(frontier, &vertexAtDepth{
				Vertex: v,
				Depth:  current.Depth + 1,
			})
		}
	}

	return nil
}
```

[Tarjan's algorithm](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/tarjan.go) (some stack operations are omitted):

```go
// StronglyConnected returns the list of strongly connected components
// within the Graph g. This information is primarily used by this package
// for cycle detection, but strongly connected components have widespread
// use.
func StronglyConnected(g *Graph) [][]Vertex {
	vs := g.Vertices()
	acct := sccAcct{
		NextIndex:   1,
		VertexIndex: make(map[Vertex]int, len(vs)),
	}
	for _, v := range vs {
		// Recurse on any non-visited nodes
		if acct.VertexIndex[v] == 0 {
			stronglyConnected(&acct, g, v)
		}
	}
	return acct.SCC
}

func stronglyConnected(acct *sccAcct, g *Graph, v Vertex) int {
	// Initial vertex visit
	index := acct.visit(v)
	minIdx := index

	for _, raw := range g.downEdgesNoCopy(v) {
		target := raw.(Vertex)
		targetIdx := acct.VertexIndex[target]

		// Recurse on successor if not yet visited
		if targetIdx == 0 {
			minIdx = min(minIdx, stronglyConnected(acct, g, target))
		} else if acct.inStack(target) {
			// Check if the vertex is in the stack
			minIdx = min(minIdx, targetIdx)
		}
	}

	// Pop the strongly connected components off the stack if
	// this is a root vertex
	if index == minIdx {
		var scc []Vertex
		for {
			v2 := acct.pop()
			scc = append(scc, v2)
			if v2 == v {
				break
			}
		}

		acct.SCC = append(acct.SCC, scc)
	}

	return minIdx
}

func min(a, b int) int {
	if a <= b {
		return a
	}
	return b
}

// sccAcct is used ot pass around accounting information for
// the StronglyConnectedComponents algorithm
type sccAcct struct {
	NextIndex   int
	VertexIndex map[Vertex]int
	Stack       []Vertex
	SCC         [][]Vertex
}

// visit assigns an index and pushes a vertex onto the stack
func (s *sccAcct) visit(v Vertex) int {
	idx := s.NextIndex
	s.VertexIndex[v] = idx
	s.NextIndex++
	s.push(v)
	return idx
}

// ...
```

## Testing

The test coverage is quite comprehensive.

[Finding the root node](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag_test.go#L23-L36):
```go
func TestAcyclicGraphRoot(t *testing.T) {
	var g AcyclicGraph
	g.Add(1)
	g.Add(2)
	g.Add(3)
	g.Connect(BasicEdge(3, 2))
	g.Connect(BasicEdge(3, 1))

	if root, err := g.Root(); err != nil {
		t.Fatalf("err: %s", err)
	} else if root != 3 {
		t.Fatalf("bad: %#v", root)
	}
}

// ...
func TestAcyclicGraphRoot_cycle(t *testing.T) {
    // ...
}

// ...
func TestAcyclicGraphRoot_multiple(t *testing.T) {
    // ...
}
```

See more tests in [dag_test.go](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag_test.go), [graph_test.go](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph_test.go), [tarjan_test.go](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/tarjan_test.go).

## Observations

* Methods like [`Graph#UpEdges`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph.go#L171) return copies. To avoid copying when it's unnecessary, there are methods like [`Graph#upEdgesNoCopy`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph.go#L191).
* There are two almost identical methods to walk a graph: "down", [`DepthFirstWalk`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L184), and "up", [`ReverseDepthFirstWalk`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L266). There's no common method to share code between them.
* There's a custom [`Set`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/set.go) class on top of golang's maps to store vertices and edges.

## References

* [GitHub Repo](https://github.com/hashicorp/terraform)
* [Product website](https://www.terraform.io/)
* [Documentation](https://www.terraform.io/docs/)
* [Applying Graph Theory to Infrastructure as Code (Video)](https://www.youtube.com/watch?v=Ce3RNfRbdZ0)

## Copyright notice

Terraform is licensed under the [Mozilla Public License 2.0](https://github.com/hashicorp/terraform/blob/main/LICENSE).

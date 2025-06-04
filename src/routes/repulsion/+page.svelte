<script lang="ts">
	import { onMount } from 'svelte';
	import * as d3 from 'd3';

	interface Node {
		id: string;
		x: number;
		y: number;
		radius: number;
		label: string;
	}

	interface Edge {
		id: string;
		source: string;
		target: string;
		name: string;
	}

	let canvasContainer: HTMLDivElement;
	let svg: d3.Selection<SVGSVGElement, unknown, null, undefined>;
	let g: d3.Selection<SVGGElement, unknown, null, undefined>;
	let zoom: d3.ZoomBehavior<SVGSVGElement, unknown>;
	
	let nodes: Node[] = [];
	let edges: Edge[] = [];
	let nodeCounter = 0;
	let edgeCounter = 0;

	const canvasWidth = 800;
	const canvasHeight = 600;
	const nodeRadius = 20;

	// Edge creation state
	let isDraggingForEdge = false;
	let dragSourceNode: Node | null = null;
	let tempEdgeLine: d3.Selection<SVGLineElement, unknown, null, undefined> | null = null;

	// Force simulation parameters
	const repelDistance = 80; // Distance at which nodes start repelling
	const repelStrength = 0.3; // Strength of repulsion force
	const maxRecursionDepth = 100; // Limit recursion to prevent performance issues

	// Reactive statement to re-render when nodes or edges change
	$: if (svg && (nodes || edges)) {
		render();
	}

	onMount(() => {
		// For simulation test: spawn 1000 nodes connected to a central node
		if (nodes.length === 0) {
			const centerNode: Node = {
				id: 'node-0',
				x: canvasWidth / 2,
				y: canvasHeight / 2,
				radius: nodeRadius,
				label: 'Center'
			};
			let nNodes = 100;
			nodes = [centerNode];
			nodeCounter = 0;
			const newNodes: Node[] = [];
			const newEdges: Edge[] = [];
			for (let i = 1; i <= nNodes; i++) {
				const angle = (2 * Math.PI * i) / nNodes;
				const x = canvasWidth / 2 + 250 * Math.cos(angle);
				const y = canvasHeight / 2 + 250 * Math.sin(angle);
				const node: Node = {
					id: `node-${i}`,
					x,
					y,
					radius: nodeRadius,
					label: `Node ${i}`
				};
				newNodes.push(node);
				newEdges.push({
					id: `edge-${i}`,
					source: centerNode.id,
					target: node.id,
					name: `Edge ${i}`
				});
			}
			nodes = [centerNode, ...newNodes];
			edges = newEdges;
			nodeCounter = nNodes;
			edgeCounter = nNodes;
		}
		initializeCanvas();
	});

	function initializeCanvas() {
		// Create SVG
		svg = d3.select(canvasContainer)
			.append('svg')
			.attr('width', canvasWidth)
			.attr('height', canvasHeight)
			.style('border', '2px solid #333')
			.style('background-color', '#fafafa');

		// Create main group for zoom/pan transformations
		g = svg.append('g');

		// Add arrowhead marker definition
		const defs = svg.append('defs');
		
		// Regular arrowhead for permanent edges
		defs.append('marker')
			.attr('id', 'arrowhead')
			.attr('viewBox', '0 -5 10 10')
			.attr('refX', 8)
			.attr('refY', 0)
			.attr('markerWidth', 6)
			.attr('markerHeight', 6)
			.attr('orient', 'auto')
			.append('path')
			.attr('d', 'M0,-5L10,0L0,5')
			.attr('fill', '#666');

		// Temporary arrowhead for edge creation (same as regular)
		defs.append('marker')
			.attr('id', 'temp-arrowhead')
			.attr('viewBox', '0 -5 10 10')
			.attr('refX', 8)
			.attr('refY', 0)
			.attr('markerWidth', 6)
			.attr('markerHeight', 6)
			.attr('orient', 'auto')
			.append('path')
			.attr('d', 'M0,-5L10,0L0,5')
			.attr('fill', '#666');

		// Set up zoom behavior
		zoom = d3.zoom<SVGSVGElement, unknown>()
			.scaleExtent([0.1, 5])
			.on('zoom', handleZoom);

		svg.call(zoom);

		// Handle canvas clicks for adding nodes
		svg.on('click', handleCanvasClick);

		// Handle mouse move for temporary edge line
		svg.on('mousemove', handleMouseMove);

		// Prevent context menu on right-click
		svg.on('contextmenu', (event) => {
			event.preventDefault();
		});

		// Initial render
		render();
	}

	function handleZoom(event: d3.D3ZoomEvent<SVGSVGElement, unknown>) {
		g.attr('transform', event.transform.toString());
	}

	function handleCanvasClick(event: MouseEvent) {
		// Prevent adding nodes when dragging or if the event was handled by a node
		if (event.defaultPrevented) return;
		
		// Check if the click target is the SVG itself (not a node)
		if (event.target !== svg.node()) return;

		const [x, y] = d3.pointer(event, g.node());
		addNode(x, y);
	}

	function handleMouseMove(event: MouseEvent) {
		if (isDraggingForEdge && dragSourceNode && tempEdgeLine) {
			const [x, y] = d3.pointer(event, g.node());
			
			// Calculate proper start and end positions accounting for node radius
			const dx = x - dragSourceNode.x;
			const dy = y - dragSourceNode.y;
			const distance = Math.sqrt(dx * dx + dy * dy);
			
			if (distance > 0) {
				const unitX = dx / distance;
				const unitY = dy / distance;
				
				// Start from edge of source node
				const startX = dragSourceNode.x + unitX * nodeRadius;
				const startY = dragSourceNode.y + unitY * nodeRadius;
				
				// End at cursor position
				tempEdgeLine
					.attr('x1', startX)
					.attr('y1', startY)
					.attr('x2', x)
					.attr('y2', y);
			}
		}
	}

	function addNode(x: number, y: number) {
		const newNode: Node = {
			id: `node-${++nodeCounter}`,
			x,
			y,
			radius: nodeRadius,
			label: `Node ${nodeCounter}`
		};

		nodes = [...nodes, newNode];
	}

	// Remove canvas boundaries: allow nodes to be placed anywhere
	// Remove boundary checks in applyRepulsionForce
	function applyRepulsionForce(draggedNode: Node, depth: number = 0, forceMultiplier: number = 1) {
		if (depth >= maxRecursionDepth) return;
		
		// Check if this node is connected to others (to skip repulsion)
		const connectedNodeIds = new Set<string>();
		edges.forEach(edge => {
			if (edge.source === draggedNode.id) connectedNodeIds.add(edge.target);
			if (edge.target === draggedNode.id) connectedNodeIds.add(edge.source);
		});

		const pushedNodes: Node[] = []; // Track which nodes were moved for recursive calls

		nodes.forEach(otherNode => {

			const dx = otherNode.x - draggedNode.x;
			const dy = otherNode.y - draggedNode.y;
			const distance = Math.sqrt(dx * dx + dy * dy);

			// Apply repulsion if nodes are too close
			if (distance < repelDistance && distance > 0) {
				const force = repelStrength * forceMultiplier * (repelDistance - distance) / distance;
				const forceX = dx * force;
				const forceY = dy * force;

				// Store original position to check if node actually moved
				const originalX = otherNode.x;
				const originalY = otherNode.y;

				// Move the other node away
				otherNode.x += forceX;
				otherNode.y += forceY;

				// No boundary checks here

				// If the node moved significantly, add it to the list for recursive repulsion
				const movedDistance = Math.sqrt((otherNode.x - originalX) ** 2 + (otherNode.y - originalY) ** 2);
				if (movedDistance > 1) { // Only recurse if node moved more than 1 pixel
					pushedNodes.push(otherNode);
				}
			}
		});

		// Recursively apply forces from pushed nodes (with reduced strength)
		pushedNodes.forEach(pushedNode => {
			// Apply recursive repulsion with reduced strength to prevent explosive forces
			applyRepulsionForce(pushedNode, depth + 1, forceMultiplier * 0.6); // 60% of current strength
		});

		// Update the nodes array to trigger reactivity (only on the initial call)
		if (depth === 0) {
			nodes = [...nodes];
		}
	}

	function handleDragStart(event: d3.D3DragEvent<SVGGElement, Node, Node>, d: Node) {
		// Stop propagation to prevent canvas pan/zoom during node drag
		event.sourceEvent.stopPropagation();
		console.log('Drag start on node:', d.label);
		// Prevent default browser behavior (text selection, etc.)
		event.sourceEvent.preventDefault();
		
		// Check if this is a right-click for edge creation
		if (event.sourceEvent.button === 2) {
			console.log('Starting edge creation from:', d.label);
			isDraggingForEdge = true;
			dragSourceNode = d;
			
			// Create temporary edge line with arrow (using regular arrow style)
			tempEdgeLine = g.append('line')
				.attr('class', 'temp-edge')
				.attr('stroke', '#666')
				.attr('stroke-width', 2)
				.attr('marker-end', 'url(#arrowhead)')
				.attr('x1', d.x + nodeRadius)
				.attr('y1', d.y)
				.attr('x2', d.x + nodeRadius + 10)
				.attr('y2', d.y);
			
			// Add visual feedback for edge creation
			d3.select(event.currentTarget).select('circle')
				.attr('stroke-width', 4)
				.attr('stroke', '#ff6b6b');
		} else {
			// Regular node drag
			isDraggingForEdge = false;
			
			// Add visual feedback for node drag
			d3.select(event.currentTarget).select('circle')
				.attr('stroke-width', 4)
				.attr('stroke', '#ff6b6b');
		}
	}

	function handleDrag(event: d3.D3DragEvent<SVGGElement, Node, Node>, d: Node) {
		if (!isDraggingForEdge) {
			// Regular node dragging
			d.x = event.x;
			d.y = event.y;
			
			// Apply repulsion force to nearby nodes
			applyRepulsionForce(d);
			
			// Trigger Svelte reactivity by creating a new array
			nodes = nodes.map(node => node.id === d.id ? { ...node, x: d.x, y: d.y } : node);
		} else if (isDraggingForEdge && dragSourceNode && tempEdgeLine) {
			// Update temporary edge line during drag
			const [x, y] = [event.x, event.y];
			
			// Calculate proper start and end positions accounting for node radius
			const dx = x - dragSourceNode.x;
			const dy = y - dragSourceNode.y;
			const distance = Math.sqrt(dx * dx + dy * dy);
			
			if (distance > 0) {
				const unitX = dx / distance;
				const unitY = dy / distance;
				
				// Start from edge of source node
				const startX = dragSourceNode.x + unitX * nodeRadius;
				const startY = dragSourceNode.y + unitY * nodeRadius;
				
				// End at current drag position
				tempEdgeLine
					.attr('x1', startX)
					.attr('y1', startY)
					.attr('x2', x)
					.attr('y2', y);
			}
		}
	}

	function handleDragEnd(event: d3.D3DragEvent<SVGGElement, Node, Node>, d: Node) {
		console.log('Drag end - isDraggingForEdge:', isDraggingForEdge);
		
		// Remove visual feedback
		d3.select(event.currentTarget).select('circle')
			.attr('stroke-width', 2)
			.attr('stroke', '#fff');

		if (isDraggingForEdge) {
			// Check if we ended on a different node
			const targetElement = document.elementFromPoint(event.sourceEvent.clientX, event.sourceEvent.clientY);
			const targetNode = findNodeFromElement(targetElement);
			
			console.log('Target element:', targetElement, 'Target node:', targetNode);
			
			if (targetNode && targetNode.id !== dragSourceNode?.id) {
				console.log('Creating edge from', dragSourceNode?.label, 'to', targetNode.label);
				// Create edge between source and target
				createEdge(dragSourceNode!, targetNode);
			}
			
			// Clean up temporary edge line
			if (tempEdgeLine) {
				tempEdgeLine.remove();
				tempEdgeLine = null;
			}
			
			// Reset edge creation state
			isDraggingForEdge = false;
			dragSourceNode = null;
		}
	}

	function findNodeFromElement(element: Element | null): Node | null {
		if (!element) return null;
		
		// Walk up the DOM tree to find a node group
		let current = element;
		while (current && current !== svg.node()) {
			if (current.classList && current.classList.contains('node')) {
				// Get the node data from D3
				const nodeData = d3.select(current).datum() as Node;
				return nodeData;
			}
			current = current.parentElement;
		}
		return null;
	}

	function createEdge(source: Node, target: Node) {
		const edgeName = prompt(`Enter edge name (from ${source.label} to ${target.label}):`);
		if (edgeName) {
			const newEdge: Edge = {
				id: `edge-${++edgeCounter}`,
				source: source.id,
				target: target.id,
				name: edgeName
			};
			
			edges = [...edges, newEdge];
		}
	}

	function deleteNode(nodeToDelete: Node) {
		console.log('Deleting node:', nodeToDelete.label);
		
		// Remove all edges connected to this node
		edges = edges.filter(edge => 
			edge.source !== nodeToDelete.id && edge.target !== nodeToDelete.id
		);
		
		// Remove the node
		nodes = nodes.filter(node => node.id !== nodeToDelete.id);
	}

	function deleteEdge(edgeToDelete: Edge) {
		console.log('Deleting edge:', edgeToDelete.name);
		
		// Remove the edge
		edges = edges.filter(edge => edge.id !== edgeToDelete.id);
	}

	function render() {
		// Render nodes
		const nodeSelection = g.selectAll<SVGGElement, Node>('.node')
			.data(nodes, d => d.id);

		const nodeEnter = nodeSelection.enter()
			.append('g')
			.attr('class', 'node')
			.attr('transform', d => `translate(${d.x}, ${d.y})`);

		nodeEnter.append('circle')
			.attr('r', d => d.radius)
			.attr('fill', '#4285f4')
			.attr('stroke', '#fff')
			.attr('stroke-width', 2);

		nodeEnter.append('text')
			.attr('text-anchor', 'middle')
			.attr('dy', '.35em')
			.attr('fill', 'white')
			.attr('font-size', '12px')
			.attr('font-weight', 'bold')
			.text(d => d.label);

		// Merge enter and update selections
		const nodeMerge = nodeEnter.merge(nodeSelection);

		// Add drag behavior to all nodes
		const drag = d3.drag<SVGGElement, Node>()
			.filter((event) => {
				// Allow both left-click (button 0) and right-click (button 2)
				return event.button === 0 || event.button === 2;
			})
			.on('start', handleDragStart)
			.on('drag', handleDrag)
			.on('end', handleDragEnd);

		nodeMerge.call(drag);

		// Add middle-click handler for node deletion
		nodeMerge.on('mousedown', (event, d) => {
			if (event.button === 1) { // Middle mouse button
				event.preventDefault();
				event.stopPropagation();
				deleteNode(d);
			}
		});

		// Update positions for existing nodes
		nodeMerge.attr('transform', d => `translate(${d.x}, ${d.y})`);

		nodeSelection.exit().remove();

		// Render edges
		const edgeSelection = g.selectAll<SVGGElement, Edge>('.edge')
			.data(edges, d => d.id);

		const edgeEnter = edgeSelection.enter()
			.append('g')
			.attr('class', 'edge');

		// Add the main edge line
		edgeEnter.append('line')
			.attr('stroke', '#666')
			.attr('stroke-width', 2)
			.attr('marker-end', 'url(#arrowhead)');

		// Add edge label background
		edgeEnter.append('rect')
			.attr('fill', 'white')
			.attr('stroke', '#666')
			.attr('stroke-width', 1)
			.attr('rx', 3);

		// Add edge label text
		edgeEnter.append('text')
			.attr('text-anchor', 'middle')
			.attr('dy', '.35em')
			.attr('font-size', '10px')
			.attr('fill', '#333')
			.text(d => d.name);

		// Update all edges (enter + existing)
		const edgeMerge = edgeEnter.merge(edgeSelection);
		
		// Add middle-click handler for edge deletion
		edgeMerge.on('mousedown', (event, d) => {
			if (event.button === 1) { // Middle mouse button
				event.preventDefault();
				event.stopPropagation();
				deleteEdge(d);
			}
		});
		
		edgeMerge.each(function(d) {
			const sourceNode = nodes.find(n => n.id === d.source);
			const targetNode = nodes.find(n => n.id === d.target);
			
			if (sourceNode && targetNode) {
				const edgeGroup = d3.select(this);
				
				// Calculate edge positions (accounting for node radius)
				const dx = targetNode.x - sourceNode.x;
				const dy = targetNode.y - sourceNode.y;
				const distance = Math.sqrt(dx * dx + dy * dy);
				
				if (distance > 0) {
					const unitX = dx / distance;
					const unitY = dy / distance;
					
					const startX = sourceNode.x + unitX * nodeRadius;
					const startY = sourceNode.y + unitY * nodeRadius;
					const endX = targetNode.x - unitX * nodeRadius;
					const endY = targetNode.y - unitY * nodeRadius;
					
					// Update line position
					edgeGroup.select('line')
						.attr('x1', startX)
						.attr('y1', startY)
						.attr('x2', endX)
						.attr('y2', endY);
					
					// Update label position (middle of edge)
					const midX = (startX + endX) / 2;
					const midY = (startY + endY) / 2;
					
					const text = edgeGroup.select('text');
					const textBBox = (text.node() as SVGTextElement)?.getBBox();
					
					if (textBBox) {
						edgeGroup.select('rect')
							.attr('x', midX - textBBox.width / 2 - 2)
							.attr('y', midY - textBBox.height / 2 - 1)
							.attr('width', textBBox.width + 4)
							.attr('height', textBBox.height + 2);
					}
					
					text.attr('x', midX)
						.attr('y', midY);
				}
			}
		});

		edgeSelection.exit().remove();
	}
</script>

<svelte:head>
	<title>Graph Database Editor</title>
	<meta name="description" content="Graph database WYSIWYG editor" />
</svelte:head>

<section>
	<h1>Graph Database Editor</h1>
	<p>Click on canvas to add nodes. Drag nodes to move them. Right-click and drag from one node to another to create edges. Middle-click on nodes or edges to delete them.</p>
	
	<div class="canvas-container" bind:this={canvasContainer}>
		<!-- D3 SVG will be inserted here -->
	</div>
</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		padding: 2rem;
		min-height: 100vh;
	}

	h1 {
		margin-bottom: 1rem;
		color: #333;
		font-size: 2rem;
		text-align: center;
	}

	p {
		margin-bottom: 2rem;
		color: #666;
		text-align: center;
		font-size: 1.1rem;
	}

	.canvas-container {
		display: flex;
		justify-content: center;
		align-items: center;
		padding: 1rem;
		background: #f5f5f5;
		border-radius: 8px;
		box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
	}

	:global(.node) {
		cursor: grab;
		user-select: none;
		-webkit-user-select: none;
		-moz-user-select: none;
		-ms-user-select: none;
	}

	:global(.node:active) {
		cursor: grabbing;
	}

	:global(.node:hover circle) {
		fill: #1a73e8 !important;
	}

	:global(.node text) {
		pointer-events: none;
		user-select: none;
		-webkit-user-select: none;
		-moz-user-select: none;
		-ms-user-select: none;
	}

	:global(.edge) {
		cursor: pointer;
	}

	:global(.edge:hover) {
		opacity: 0.8;
	}

	:global(.temp-edge) {
		pointer-events: none;
	}
</style>

<script lang="ts">
import { onMount, onDestroy } from 'svelte';
import * as d3 from 'd3';

// Node and Edge interfaces (compatible with both modes)
interface Node extends d3.SimulationNodeDatum {
	id: string;
	x: number;
	y: number;
	radius: number;
	label: string;
}
interface Edge extends d3.SimulationLinkDatum<Node> {
	id: string;
	source: string | Node;
	target: string | Node;
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

// Mode: 'force' (default) or 'manual'
let mode: 'force' | 'manual' = 'force';

// --- Shared state for both modes ---
let isDraggingForEdge = false;
let dragSourceNode: Node | null = null;
let tempEdgeLine: d3.Selection<SVGLineElement, unknown, null, undefined> | null = null;

// --- Force simulation state ---
let simulation: d3.Simulation<Node, Edge>;
let animationFrameId: number | null = null;
let isDraggingNode = false;

// --- Manual repulsion state ---
const repelDistance = 80;
const repelStrength = 0.3;
const maxRecursionDepth = 100;

// --- Initialization ---
$: if (svg && (nodes || edges)) {
	render();
}

onMount(() => {
	initializeCanvas();
	if (mode === 'force') initializeSimulation();
	spawnTestGraph();
	if (mode === 'force') restartSimulation();
});

onDestroy(() => {
	if (simulation) simulation.stop();
	if (animationFrameId) cancelAnimationFrame(animationFrameId);
});

function spawnTestGraph() {
	const nNodes = 100;
	const centerNode: Node = {
		id: 'node-0',
		x: canvasWidth / 2,
		y: canvasHeight / 2,
		radius: nodeRadius,
		label: 'Center'
	};
	nodes = [centerNode];
	edges = [];
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
		nodes.push(node);
		edges.push({
			id: `edge-${i}`,
			source: centerNode.id,
			target: node.id,
			name: `Edge ${i}`
		});
	}
	nodeCounter = nNodes;
	edgeCounter = nNodes;
}

function initializeCanvas() {
	svg = d3.select(canvasContainer)
		.append('svg')
		.attr('width', canvasWidth)
		.attr('height', canvasHeight)
		.style('border', '2px solid #333')
		.style('background-color', '#fafafa');
	g = svg.append('g');
	const defs = svg.append('defs');
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
	zoom = d3.zoom<SVGSVGElement, unknown>()
		.scaleExtent([0.1, 5])
		.on('zoom', handleZoom);
	svg.call(zoom);
	svg.on('click', handleCanvasClick);
	svg.on('mousemove', handleMouseMove);
	svg.on('contextmenu', (event) => event.preventDefault());
	render();
}

function initializeSimulation() {
	simulation = d3.forceSimulation<Node>(nodes)
		.force('charge', d3.forceManyBody().strength(-300).distanceMax(80).distanceMin(5))
		.force('collision', d3.forceCollide().radius(nodeRadius + 2).strength(0.7))
		.alphaDecay(0.25)
		.velocityDecay(0.7)
		.on('tick', onSimulationTick);
	simulation.stop();
}

function onSimulationTick() {
	if (!animationFrameId) {
		animationFrameId = requestAnimationFrame(() => {
			animationFrameId = null;
			render();
		});
	}
}

function restartSimulation() {
	if (simulation) {
		simulation.nodes(nodes);
		simulation.alpha(0.3).restart();
		setTimeout(() => simulation.stop(), 400);
	}
}

function switchMode(newMode: 'force' | 'manual') {
	if (mode === newMode) return;
	mode = newMode;
	if (mode === 'force') {
		initializeSimulation();
		restartSimulation();
	} else {
		if (simulation) simulation.stop();
	}
	render();
}

function handleZoom(event: d3.D3ZoomEvent<SVGSVGElement, unknown>) {
	g.attr('transform', event.transform.toString());
}

function handleCanvasClick(event: MouseEvent) {
	if (event.defaultPrevented) return;
	if (event.target !== svg.node()) return;
	const [x, y] = d3.pointer(event, g.node());
	addNode(x, y);
}

function handleMouseMove(event: MouseEvent) {
	if (isDraggingForEdge && dragSourceNode && tempEdgeLine) {
		const [x, y] = d3.pointer(event, g.node());
		const dx = x - dragSourceNode.x;
		const dy = y - dragSourceNode.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		if (distance > 0) {
			const unitX = dx / distance;
			const unitY = dy / distance;
			const startX = dragSourceNode.x + unitX * nodeRadius;
			const startY = dragSourceNode.y + unitY * nodeRadius;
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
	if (mode === 'force') restartSimulation();
}

function applyRepulsionForce(draggedNode: Node, depth: number = 0, forceMultiplier: number = 1) {
	if (depth >= maxRecursionDepth) return;
	const connectedNodeIds = new Set<string>();
	edges.forEach(edge => {
		if (edge.source === draggedNode.id) connectedNodeIds.add(edge.target as string);
		if (edge.target === draggedNode.id) connectedNodeIds.add(edge.source as string);
	});
	const pushedNodes: Node[] = [];
	nodes.forEach(otherNode => {
		const dx = otherNode.x - draggedNode.x;
		const dy = otherNode.y - draggedNode.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		if (distance < repelDistance && distance > 0) {
			const force = repelStrength * forceMultiplier * (repelDistance - distance) / distance;
			const forceX = dx * force;
			const forceY = dy * force;
			const originalX = otherNode.x;
			const originalY = otherNode.y;
			otherNode.x += forceX;
			otherNode.y += forceY;
			const movedDistance = Math.sqrt((otherNode.x - originalX) ** 2 + (otherNode.y - originalY) ** 2);
			if (movedDistance > 1) pushedNodes.push(otherNode);
		}
	});
	pushedNodes.forEach(pushedNode => {
		applyRepulsionForce(pushedNode, depth + 1, forceMultiplier * 0.6);
	});
	if (depth === 0) nodes = [...nodes];
}

function handleDragStart(event: d3.D3DragEvent<SVGGElement, Node, Node>, d: Node) {
	event.sourceEvent.stopPropagation();
	event.sourceEvent.preventDefault();
	isDraggingNode = true;
	if (event.sourceEvent.button === 2) {
		isDraggingForEdge = true;
		dragSourceNode = d;
		tempEdgeLine = g.append('line')
			.attr('class', 'temp-edge')
			.attr('stroke', '#666')
			.attr('stroke-width', 2)
			.attr('marker-end', 'url(#arrowhead)')
			.attr('x1', d.x + nodeRadius)
			.attr('y1', d.y)
			.attr('x2', d.x + nodeRadius + 10)
			.attr('y2', d.y);
		d3.select(event.currentTarget).select('circle')
			.attr('stroke-width', 4)
			.attr('stroke', '#ff6b6b');
	} else {
		isDraggingForEdge = false;
		if (mode === 'force') {
			d.fx = d.x;
			d.fy = d.y;
			if (simulation) simulation.alphaTarget(0.3).restart();
		} else {
			// manual mode: just highlight
		}
		d3.select(event.currentTarget).select('circle')
			.attr('stroke-width', 4)
			.attr('stroke', '#ff6b6b');
	}
}

function handleDrag(event: d3.D3DragEvent<SVGGElement, Node, Node>, d: Node) {
	if (!isDraggingForEdge) {
		if (mode === 'force') {
			d.fx = event.x;
			d.fy = event.y;
			d.x = event.x;
			d.y = event.y;
		} else {
			d.x = event.x;
			d.y = event.y;
			applyRepulsionForce(d);
			nodes = nodes.map(node => node.id === d.id ? { ...node, x: d.x, y: d.y } : node);
		}
	} else if (isDraggingForEdge && dragSourceNode && tempEdgeLine) {
		const [x, y] = [event.x, event.y];
		const dx = x - dragSourceNode.x;
		const dy = y - dragSourceNode.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		if (distance > 0) {
			const unitX = dx / distance;
			const unitY = dy / distance;
			const startX = dragSourceNode.x + unitX * nodeRadius;
			const startY = dragSourceNode.y + unitY * nodeRadius;
			tempEdgeLine
				.attr('x1', startX)
				.attr('y1', startY)
				.attr('x2', x)
				.attr('y2', y);
		}
	}
}

function handleDragEnd(event: d3.D3DragEvent<SVGGElement, Node, Node>, d: Node) {
	isDraggingNode = false;
	d3.select(event.currentTarget).select('circle')
		.attr('stroke-width', 2)
		.attr('stroke', '#fff');
	if (isDraggingForEdge) {
		const targetElement = document.elementFromPoint(event.sourceEvent.clientX, event.sourceEvent.clientY);
		const targetNode = findNodeFromElement(targetElement);
		if (targetNode && targetNode.id !== dragSourceNode?.id) {
			createEdge(dragSourceNode!, targetNode);
		}
		if (tempEdgeLine) {
			tempEdgeLine.remove();
			tempEdgeLine = null;
		}
		isDraggingForEdge = false;
		dragSourceNode = null;
	} else {
		if (mode === 'force') {
			d.fx = null;
			d.fy = null;
			if (simulation) {
				simulation.alphaTarget(0);
				setTimeout(() => simulation.stop(), 200);
			}
		}
	}
}

function findNodeFromElement(element: Element | null): Node | null {
	if (!element) return null;
	let current = element;
	while (current && current !== svg.node()) {
		if (current.classList && current.classList.contains('node')) {
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
		if (mode === 'force') restartSimulation();
	}
}

function deleteNode(nodeToDelete: Node) {
	edges = edges.filter(edge => {
		const sourceId = typeof edge.source === 'string' ? edge.source : edge.source.id;
		const targetId = typeof edge.target === 'string' ? edge.target : edge.target.id;
		return sourceId !== nodeToDelete.id && targetId !== nodeToDelete.id;
	});
	nodes = nodes.filter(node => node.id !== nodeToDelete.id);
	if (mode === 'force') restartSimulation();
}

function deleteEdge(edgeToDelete: Edge) {
	edges = edges.filter(edge => edge.id !== edgeToDelete.id);
	if (mode === 'force') restartSimulation();
}

function render() {
	// ...existing code for rendering nodes and edges, same as in your current editors...
	// (copy from either file, as they are nearly identical)
}
</script>

<svelte:head>
	<title>Graph Database Editor (Combined)</title>
	<meta name="description" content="Graph database WYSIWYG editor with force/manual modes" />
</svelte:head>

<section>
	<h1>Graph Database Editor (Force/Manual Switch)</h1>
	<p>
		Click on canvas to add nodes. Drag nodes to move them. Right-click and drag from one node to another to create edges. Middle-click on nodes or edges to delete them.
	</p>
	<div style="margin-bottom:1rem;">
		<label><input type="radio" bind:group={mode} value="force" on:change={() => switchMode('force')}> Force Simulation (default)</label>
		<label style="margin-left:2em;"><input type="radio" bind:group={mode} value="manual" on:change={() => switchMode('manual')}> Manual Repulsion</label>
	</div>
	<div class="canvas-container" bind:this={canvasContainer}>
		<!-- D3 SVG will be inserted here -->
	</div>
</section>

<style>
/* ...existing styles from your editors... */
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
	margin-bottom: 1rem;
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

<script lang="ts">
import { onMount, onDestroy } from 'svelte';
import { browser } from '$app/environment';
import * as d3 from 'd3';

// Node and Edge interfaces (compatible with both modes)
interface Node extends d3.SimulationNodeDatum {
	id: string;
	x: number;
	y: number;
	radius: number;
	label: string;
	color: string;
	// Added for arbitrary text properties (e.g., URL, description, etc.)
	properties: Record<string, string>;
}
interface Edge extends d3.SimulationLinkDatum<Node> {
	id: string;
	source: string | Node;
	target: string | Node;
	name: string;
	// Added for arbitrary text properties (e.g., URL, description, etc.)
	properties: Record<string, string>;
}

let canvasContainer: HTMLDivElement;
let svg: d3.Selection<SVGSVGElement, unknown, null, undefined>;
let g: d3.Selection<SVGGElement, unknown, null, undefined>;
let zoom: d3.ZoomBehavior<SVGSVGElement, unknown>;

let nodes: Node[] = [];
let edges: Edge[] = [];
let nodeCounter = 0;
let edgeCounter = 0;

const canvasWidth = 1700;
const canvasHeight = 700;
const minNodeRadius = 15; // Minimum radius for readability
const nodePadding = 8; // Padding around text inside the circle

// Mode: 'force' (default) or 'manual'
let mode: 'force' | 'manual' = 'force';

// --- Selection Box State ---
let selection: d3.Selection<SVGPathElement, unknown, null, undefined>;
let isSelecting = false;
let selectionStart: [number, number] = [0, 0];
// --- Shared state for both modes ---
let isDraggingForEdge = false;
let dragSourceNode: Node | null = null;
let tempEdgeLine: d3.Selection<SVGLineElement, unknown, null, undefined> | null = null;
let justCompletedEdgeCreation = false;

// --- Force simulation state ---
let simulation: d3.Simulation<Node, Edge>;
let animationFrameId: number | null = null;
let isDraggingNode = false;

// --- Click vs Drag detection ---
let dragStartPosition: { x: number; y: number } | null = null;
let draggedNode: Node | null = null;
const clickThreshold = 3; // pixels - if movement is less than this, consider it a click

// --- Multi-select drag state ---
let dragOffsets: Map<string, { dx: number; dy: number }> = new Map();

// --- Manual repulsion state ---
const repelDistance = 80;
const repelStrength = 0.3;

// --- Text Measurement Function ---
function rect(x: number, y: number, w: number, h: number) {
	return "M" + [x, y] + " l" + [w, 0] + " l" + [0, h] + " l" + [-w, 0] + "z";
}

function calculateNodeRadius(text: string): number {
	// Create a temporary text element to measure text dimensions
	const tempText = svg.append('text')
		.attr('font-size', '12px')
		.attr('font-weight', 'bold')
		.text(text)
		.style('visibility', 'hidden');
	
	const bbox = (tempText.node() as SVGTextElement).getBBox();
	tempText.remove();
	
	// Calculate radius based on text width and height with padding
	// Use the larger dimension to ensure text fits in the circle
	const textWidth = bbox.width;
	const textHeight = bbox.height;
	const maxDimension = Math.max(textWidth, textHeight);
	
	// Calculate radius: half the diagonal of the text bounding box plus padding
	// This ensures the text fits comfortably within the circle
	const requiredRadius = Math.max(
		(Math.sqrt(textWidth * textWidth + textHeight * textHeight) / 2) + nodePadding,
		minNodeRadius
	);
	
	return Math.ceil(requiredRadius);
}
const maxRecursionDepth = 100;

// --- Dialog state for renaming nodes/edges ---
let showRenameDialog = false;
let renameType: 'node' | 'edge' | null = null;
let selectedItem: Node | Edge | null = null;
let renameValue = '';

// --- Node creation state ---
let showCreateNodeDialog = false;
let pendingNodePosition: { x: number; y: number } | null = null;
let createNodeName = '';

// --- Edge creation dialog state ---
let showCreateEdgeDialog = false;
let pendingEdge: { source: Node; target: Node } | null = null;
let createEdgeName = '';

// --- Dialog state for creating edge+node ---
let showCreateEdgeAndNodeDialog = false;
let pendingEdgeSourceNode: Node | null = null;
let pendingEdgeStart: { x: number; y: number } | null = null;
let pendingEdgeEnd: { x: number; y: number } | null = null;
let newEdgeName = '';
let newNodeName = '';

// --- Selection state ---
let selectedNodeIds = new Set<string>();
let selectedEdgeIds = new Set<string>();

// --- Color palette ---
const colorPalette = [
	'#4285f4', // Blue (default)
	'#ea4335', // Red
	'#34a853', // Green
	'#fbbc05', // Yellow
	'#9c27b0', // Purple
	'#ff9800', // Orange
	'#e91e63', // Pink
	'#00bcd4', // Cyan
	'#795548', // Brown
	'#607d8b'  // Blue Grey
];

// --- Backup and Save/Load functionality ---
let autoSaveEnabled = true;
let fileHandle: any = null; // For File System Access API
let lastSaved = '';

// Auto-save to localStorage whenever graph changes
$: if (browser && autoSaveEnabled && (nodes.length > 0 || edges.length > 0)) {
	autoSaveToLocalStorage();
}

function autoSaveToLocalStorage() {
	try {
		const graphData = {
			nodes: nodes.map(n => ({ ...n, fx: undefined, fy: undefined, vx: undefined, vy: undefined })),
			edges: edges.map(e => ({ 
				...e, 
				source: typeof e.source === 'object' ? e.source.id : e.source,
				target: typeof e.target === 'object' ? e.target.id : e.target
			})),
			nodeCounter,
			edgeCounter,
			mode,
			timestamp: new Date().toISOString()
		};
		localStorage.setItem('graph-editor-autosave', JSON.stringify(graphData));
		lastSaved = new Date().toLocaleTimeString();
	} catch (error) {
		console.warn('Auto-save failed:', error);
	}
}

function loadFromLocalStorage() {
	try {
		const saved = localStorage.getItem('graph-editor-autosave');
		if (saved) {
			const graphData = JSON.parse(saved);
			nodes = graphData.nodes || [];
			edges = graphData.edges || [];
			nodeCounter = graphData.nodeCounter || 0;
			edgeCounter = graphData.edgeCounter || 0;
			mode = graphData.mode || 'force';
			lastSaved = new Date(graphData.timestamp).toLocaleTimeString();
			restartSimulation();
			console.log('Loaded graph from auto-save');
			return true;
		}
	} catch (error) {
		console.warn('Failed to load from localStorage:', error);
	}
	return false;
}

function exportGraphAsJSON() {
	const graphData = {
		nodes: nodes.map(n => ({ ...n, fx: undefined, fy: undefined, vx: undefined, vy: undefined })),
		edges: edges.map(e => ({ 
			...e, 
			source: typeof e.source === 'object' ? e.source.id : e.source,
			target: typeof e.target === 'object' ? e.target.id : e.target
		})),
		nodeCounter,
		edgeCounter,
		mode,
		exportedAt: new Date().toISOString()
	};
	return JSON.stringify(graphData, null, 2);
}

function downloadGraph() {
	const jsonData = exportGraphAsJSON();
	const blob = new Blob([jsonData], { type: 'application/json' });
	const url = URL.createObjectURL(blob);
	const a = document.createElement('a');
	a.href = url;
	a.download = `graph-${new Date().toISOString().slice(0, 19).replace(/:/g, '-')}.json`;
	document.body.appendChild(a);
	a.click();
	document.body.removeChild(a);
	URL.revokeObjectURL(url);
}

async function saveToFile() {
	if (!('showSaveFilePicker' in window)) {
		// Fallback to download for browsers without File System Access API
		downloadGraph();
		return;
	}

	try {
		if (!fileHandle) {
			fileHandle = await (window as any).showSaveFilePicker({
				suggestedName: `graph-${new Date().toISOString().slice(0, 10)}.json`,
				types: [{
					description: 'JSON files',
					accept: { 'application/json': ['.json'] }
				}]
			});
		}

		const writable = await fileHandle.createWritable();
		await writable.write(exportGraphAsJSON());
		await writable.close();
		lastSaved = new Date().toLocaleTimeString();
		console.log('Graph saved to file');		} catch (error: any) {
			if (error.name !== 'AbortError') {
				console.error('Save failed:', error);
				// Fallback to download
				downloadGraph();
			}
		}
}

async function loadFromFile() {
	if (!('showOpenFilePicker' in window)) {
		// Fallback to file input for browsers without File System Access API
		const input = document.createElement('input');
		input.type = 'file';
		input.accept = '.json';
		input.onchange = (e) => {
			const file = (e.target as HTMLInputElement).files?.[0];
			if (file) {
				const reader = new FileReader();
				reader.onload = (e) => {
					try {
						const graphData = JSON.parse(e.target?.result as string);
						loadGraphData(graphData);
					} catch (error) {
						alert('Error loading file: Invalid JSON format');
					}
				};
				reader.readAsText(file);
			}
		};
		input.click();
		return;
	}

	try {
		const [fileHandle] = await (window as any).showOpenFilePicker({
			types: [{
				description: 'JSON files',
				accept: { 'application/json': ['.json'] }
			}]
		});

		const file = await fileHandle.getFile();
		const text = await file.text();
		const graphData = JSON.parse(text);
		loadGraphData(graphData);
	} catch (error) {
		if (error.name !== 'AbortError') {
			console.error('Load failed:', error);
			alert('Error loading file');
		}
	}
}

function loadGraphData(graphData: any) {
	try {
		nodes = graphData.nodes || [];
		edges = graphData.edges || [];
		nodeCounter = graphData.nodeCounter || 0;
		edgeCounter = graphData.edgeCounter || 0;
		mode = graphData.mode || 'force';
		clearSelection();
		restartSimulation();
		console.log('Graph loaded successfully');
	} catch (error) {
		console.error('Error loading graph data:', error);
		alert('Error loading graph: Invalid data format');
	}
}

function clearGraph() {
	if (confirm('Are you sure you want to clear the entire graph? This cannot be undone.')) {
		nodes = [];
		edges = [];
		nodeCounter = 0;
		edgeCounter = 0;
		clearSelection();
		if (simulation) simulation.stop();
		render();
	}
}

function newGraph() {
	clearGraph();
	fileHandle = null; // Reset file handle for new graph
}

// --- Initialization ---
$: if (svg && (nodes || edges)) {
	render();
}

onMount(() => {
	initializeCanvas();
	if (mode === 'force') initializeSimulation();
	
	// Try to load from localStorage first, otherwise spawn test graph
	if (!loadFromLocalStorage()) {
		spawnTestGraph();
	}
	
	if (mode === 'force') restartSimulation();
	if (browser) {
		window.addEventListener('keydown', handleKeyDown);
		
		// Add global context menu handler to prevent context menu after edge creation
		const handleGlobalContextMenu = (event: MouseEvent) => {
			if (justCompletedEdgeCreation) {
				console.log('Preventing global context menu after edge creation');
				event.preventDefault();
				event.stopPropagation();
			}
		};
		document.addEventListener('contextmenu', handleGlobalContextMenu);
		
		// Store reference for cleanup
		window.globalContextMenuHandler = handleGlobalContextMenu;
	}
});

onDestroy(() => {
	if (simulation) simulation.stop();
	if (animationFrameId) cancelAnimationFrame(animationFrameId);
	if (browser) {
		window.removeEventListener('keydown', handleKeyDown);
		
		// Remove global context menu handler
		if (window.globalContextMenuHandler) {
			document.removeEventListener('contextmenu', window.globalContextMenuHandler);
		}
	}
});

function spawnTestGraph() {
	const nNodes = 100;	const centerNode: Node = {
		id: 'node-0',
		x: canvasWidth / 2,
		y: canvasHeight / 2,
		radius: calculateNodeRadius('Center'),
		label: 'Center',
		color: '#4285f4'
	};
	nodes = [centerNode];
	edges = [];
	for (let i = 1; i <= nNodes; i++) {
		const angle = (2 * Math.PI * i) / nNodes;
		const x = canvasWidth / 2 + 250 * Math.cos(angle);
		const y = canvasHeight / 2 + 250 * Math.sin(angle);		const nodeLabel = `Node ${i}`;
		const node: Node = {
			id: `node-${i}`,
			x,
			y,
			radius: calculateNodeRadius(nodeLabel),
			label: nodeLabel,
			color: '#4285f4'
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
		.style('background-color', '#fafafa');	g = svg.append('g');
	
	// Add selection path element
	selection = svg.append("path")
		.attr("class", "selection")
		.attr("visibility", "hidden");
	
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
		.filter((event) => {
			// Only allow zooming/panning with middle mouse button (button === 1) or wheel events
			// Don't preventDefault here - let D3 handle it
			return event.button === 1 || event.type === 'wheel';
		})
		.on('start', (event) => {
			// Prevent default behavior when zoom/pan starts with middle mouse
			if (event.sourceEvent && event.sourceEvent.button === 1) {
				event.sourceEvent.preventDefault();
				event.sourceEvent.stopPropagation();
			}
		})
		.on('zoom', handleZoom);
	svg.call(zoom);
	
	svg.on('mousedown.selection', handleCanvasMouseDown); // Namespaced mousedown for selection
	svg.on('mousemove.selection', handleMouseMove);     // Namespaced mousemove for selection
	svg.on('mouseup.selection', handleCanvasMouseUp);       // Namespaced mouseup for selection
	svg.on('contextmenu', handleCanvasClick); // For right-click actions (e.g., new node)

	// Add color palette
	createColorPalette();
	
	render();
}

function handleCanvasMouseDown(event: MouseEvent) {
	// Only proceed if the mousedown is directly on the SVG canvas (not on a node/edge or other element)
	// and it's a left click (button 0)
	if (event.target !== svg.node() || event.button !== 0) {
		return;
	}

	event.preventDefault(); // Prevent default browser actions like text selection
	event.stopPropagation(); // Crucially, stop the event from propagating to the zoom behavior

	isSelecting = true;
	const start = d3.pointer(event, svg.node()); // Get coordinates relative to the SVG element
	selectionStart = start;
	
	// Start the selection box
	selection.attr("d", rect(start[0], start[1], 0, 0))
		.attr("visibility", "visible");

	// If not holding Shift, clear any existing selection
	if (!event.shiftKey) {
		clearSelection(); // This function should handle re-rendering
	} else {
		render(); // If shift is held, still need to render to show the selection box potentially
	}
}

function handleCanvasMouseUp(event: MouseEvent) {
	if (!isSelecting || event.button !== 0) {
		// If not actively selecting or not a left mouse button up, reset if needed and exit
		if (isSelecting) {
			isSelecting = false;
			selection.attr("visibility", "hidden");
			render(); // Ensure selection box is hidden
		}
		return;
	}

	isSelecting = false;
	selection.attr("visibility", "hidden"); // Hide the box first

	const end = d3.pointer(event, svg.node());

	// Check if it was a click (minimal drag) vs a drag for selection
	const movedDistance = Math.sqrt(
		Math.pow(end[0] - selectionStart[0], 2) + Math.pow(end[1] - selectionStart[1], 2)
	);

	if (movedDistance < clickThreshold) {
		// It was a click on the canvas, not a drag for selection box
		if (!event.shiftKey) {
			clearSelection(); // Clears selection and re-renders
		} else {
			render(); // Re-render if shift was held, to ensure UI is up-to-date
		}
	} else {
		// It was a drag for selection box
		const selX = Math.min(selectionStart[0], end[0]);
		const selY = Math.min(selectionStart[1], end[1]);
		const selWidth = Math.abs(end[0] - selectionStart[0]);
		const selHeight = Math.abs(end[1] - selectionStart[1]);

		const newlySelectedNodeIds = new Set<string>();
		nodes.forEach(node => {
			// Convert node coordinates from g-space to svg-space
			const transform = d3.zoomTransform(svg.node()!);
			const nodeX = transform.applyX(node.x);
			const nodeY = transform.applyY(node.y);
			
			if (
				nodeX >= selX &&
				nodeX <= selX + selWidth &&
				nodeY >= selY &&
				nodeY <= selY + selHeight
			) {
				newlySelectedNodeIds.add(node.id);
			}
		});

		if (!event.shiftKey) {
			selectedNodeIds.clear();
			selectedEdgeIds.clear(); // Edges are not selected by box, but clear for consistency
			newlySelectedNodeIds.forEach(id => selectedNodeIds.add(id));
		} else {
			// Additive selection with Shift
			newlySelectedNodeIds.forEach(id => {
				if (selectedNodeIds.has(id)) {
					// Optionally, Shift+Drag could toggle, but typically it adds
					// selectedNodeIds.delete(id); // To toggle
				} else {
					selectedNodeIds.add(id);
				}
			});
		}
		selectedNodeIds = new Set(selectedNodeIds); // Trigger Svelte reactivity
		updateColorPaletteSelection();
		render();
	}
}

function createColorPalette() {
	const paletteGroup = svg.append('g')
		.attr('class', 'color-palette')
		.attr('transform', 'translate(20, 20)');
	
	// Background for palette
	paletteGroup.append('rect')
		.attr('x', -5)
		.attr('y', -5)
		.attr('width', 110)
		.attr('height', 60)
		.attr('fill', 'rgba(255, 255, 255, 0.9)')
		.attr('stroke', '#ddd')
		.attr('stroke-width', 1)
		.attr('rx', 5);
	
	// Title
	paletteGroup.append('text')
		.attr('x', 50)
		.attr('y', 10)
		.attr('text-anchor', 'middle')
		.attr('font-size', '11px')
		.attr('fill', '#666')
		.text('Node Colors');
	
	// Color swatches
	const swatches = paletteGroup.selectAll('.color-swatch')
		.data(colorPalette)
		.enter()
		.append('g')
		.attr('class', 'color-swatch')
		.style('cursor', 'pointer');
	
	swatches.append('circle')
		.attr('cx', (d, i) => 15 + (i % 5) * 20)
		.attr('cy', (d, i) => 25 + Math.floor(i / 5) * 20)
		.attr('r', 8)
		.attr('fill', d => d)
		.attr('stroke', '#333')
		.attr('stroke-width', 1);
	
	// Selection indicator (initially hidden)
	paletteGroup.append('circle')
		.attr('class', 'color-selection-indicator')
		.attr('r', 10)
		.attr('fill', 'none')
		.attr('stroke', '#000')
		.attr('stroke-width', 2)
		.style('opacity', 0);
	
	// Click handlers for color swatches
	swatches.on('click', function(event, color) {
		event.stopPropagation();
		if (selectedNodeIds.size > 0) {
			changeSelectedNodesColor(color);
			updateColorPaletteSelection();
		}
	});
}

function updateColorPaletteSelection() {
	const selectedColor = getSelectedNodeColor();
	const indicator = svg.select('.color-selection-indicator');
	
	if (selectedColor && selectedNodeIds.size > 0) {
		const colorIndex = colorPalette.indexOf(selectedColor);
		if (colorIndex !== -1) {
			const x = 15 + (colorIndex % 5) * 20;
			const y = 25 + Math.floor(colorIndex / 5) * 20;
			indicator
				.attr('cx', x)
				.attr('cy', y)
				.style('opacity', 1);
		} else {
			indicator.style('opacity', 0);
		}
	} else {
		indicator.style('opacity', 0);
	}
}

function initializeSimulation() {
	simulation = d3.forceSimulation<Node>(nodes)
		.force('charge', d3.forceManyBody().strength(-300).distanceMax(80).distanceMin(5))
		.force('collision', d3.forceCollide().radius(d => d.radius + 2).strength(0.7))
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
		simulation.force('collision', d3.forceCollide().radius(d => d.radius + 2).strength(0.7));
		simulation.alpha(0.3).restart();
		// Removed auto-stop timeout to keep simulation active
	}
}

function switchMode(newMode: 'force' | 'manual') {
	if (mode === newMode) return;
	mode = newMode;
	if (mode === 'force') {
		// Always re-create simulation from current nodes
		initializeSimulation();
		simulation.nodes(nodes);
		simulation.force('collision', d3.forceCollide().radius(d => d.radius + 2).strength(0.7));
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
	// This function now primarily handles contextmenu events on the canvas
	if (event.type === 'contextmenu') {
		// Only show create node dialog if right-click is directly on SVG background or canvas container
		if (event.target !== svg.node() && event.target !== canvasContainer) {
			// Check if the target is part of a node or edge
			if (event.target instanceof Element && (event.target.closest('.node') || event.target.closest('.edge'))) {
				return; // Do nothing if context menu is on a node or edge, let their handlers (if any) take over.
			}
		}
		
		event.preventDefault();
		if (justCompletedEdgeCreation) {
			console.log('Blocking context menu after edge creation');
			return;
		}
		const [x, y] = d3.pointer(event, g.node());
		pendingNodePosition = { x, y };
		createNodeName = ''; // Ensure input starts blank
		showCreateNodeDialog = true;
		
		if (!event.shiftKey) { // Clear selection unless shift is held
			clearSelection();
		}
	}
	// Left-click logic on canvas is now handled by handleCanvasMouseDown/Up
}

function handleMouseMove(event: MouseEvent) {
	if (isSelecting) {
		const moved = d3.pointer(event, svg.node());
		selection.attr("d", rect(selectionStart[0], selectionStart[1], moved[0] - selectionStart[0], moved[1] - selectionStart[1]));
	} else if (isDraggingForEdge && dragSourceNode && tempEdgeLine) {
		const [x, y] = d3.pointer(event, g.node());
		const dx = x - dragSourceNode.x;
		const dy = y - dragSourceNode.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		if (distance > 0) {
			const unitX = dx / distance;
			const unitY = dy / distance;
			const startX = dragSourceNode.x + unitX * dragSourceNode.radius;
			const startY = dragSourceNode.y + unitY * dragSourceNode.radius;
			tempEdgeLine
				.attr('x1', startX)
				.attr('y1', startY)
				.attr('x2', x)
				.attr('y2', y);
		}
	}
}

function addNode(x: number, y: number, label?: string) {
	const nodeLabel = label || `Node ${nodeCounter + 1}`;
	const radius = calculateNodeRadius(nodeLabel);
	
	const newNode: Node = {
		id: `node-${++nodeCounter}`,
		x,
		y,
		radius: radius,
		label: nodeLabel,
		color: '#4285f4',
		properties: {} // Always initialize properties
	};
	nodes = [...nodes, newNode];
	restartSimulation();
	return newNode;
}

function createNodeFromDialog() {
	if (pendingNodePosition && createNodeName.trim()) {
		addNode(pendingNodePosition.x, pendingNodePosition.y, createNodeName.trim());
	}
	// If no name provided, just close dialog without creating node
	closeCreateNodeDialog();
	render();
}

function closeCreateNodeDialog() {
	showCreateNodeDialog = false;
	pendingNodePosition = null;
	createNodeName = '';
}

function confirmCreateEdge() {
	if (pendingEdge && createEdgeName.trim()) {
		const newEdge: Edge = {
			id: `edge-${++edgeCounter}`,
			source: pendingEdge.source.id,
			target: pendingEdge.target.id,
			name: createEdgeName.trim(),
			properties: {} // Always initialize properties
		};
		edges = [...edges, newEdge];
		if (mode === 'force') restartSimulation();
	}
	closeCreateEdgeDialog();
}

function createEdgeFromDialog() {
	confirmCreateEdge();
	if (!createEdgeName.trim() || !pendingEdge) {
		closeCreateEdgeDialog();
		return;
	}
	edges = [
		...edges,
		{
			id: `edge-${edgeCounter++}`,
			source: pendingEdge.source.id,
			target: pendingEdge.target.id,
			name: createEdgeName,
			properties: {} // Always initialize properties
		}
	];
	closeCreateEdgeDialog();
	render();
}

function openCreateEdgeAndNodeDialog(sourceNode: Node, start: {x: number, y: number}, end: {x: number, y: number}) {
    showCreateEdgeAndNodeDialog = true;
    pendingEdgeSourceNode = sourceNode;
    pendingEdgeStart = start;
    pendingEdgeEnd = end;
    newEdgeName = '';
    newNodeName = '';
}

function closeCreateEdgeAndNodeDialog() {
    showCreateEdgeAndNodeDialog = false;
    pendingEdgeSourceNode = null;
    pendingEdgeStart = null;
    pendingEdgeEnd = null;
    newEdgeName = '';
    newNodeName = '';
}

function confirmCreateEdgeAndNode() {
    if (pendingEdgeSourceNode && pendingEdgeEnd && newEdgeName.trim() && newNodeName.trim()) {
        // Place new node at drag end position
        const x = pendingEdgeEnd.x;
        const y = pendingEdgeEnd.y;
        const label = newNodeName.trim();
        const radius = calculateNodeRadius(label);
        const newNode: Node = {
            id: `node-${++nodeCounter}`,
            x,
            y,
            radius,
            label,
            color: '#4285f4',
            properties: {} // Always initialize properties
        };
        nodes = [...nodes, newNode];
        // Create edge from source to new node
        const newEdge: Edge = {
            id: `edge-${++edgeCounter}`,
            source: pendingEdgeSourceNode.id,
            target: newNode.id,
            name: newEdgeName.trim(),
            properties: {} // Always initialize properties
        };
        edges = [...edges, newEdge];
        restartSimulation();
        render();
    }
    closeCreateEdgeAndNodeDialog();
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
		// Right-click: start edge creation
		// Prevent the context menu event that will fire after drag
		justCompletedEdgeCreation = false;
		
		isDraggingForEdge = true;
		dragSourceNode = d;
		tempEdgeLine = g.append('line')
			.attr('class', 'temp-edge')
			.attr('stroke', '#666')
			.attr('stroke-width', 2)
			.attr('marker-end', 'url(#arrowhead)')
			.attr('x1', d.x + d.radius)
			.attr('y1', d.y)
			.attr('x2', d.x + d.radius + 10)
			.attr('y2', d.y);
		d3.select(event.currentTarget).select('circle')
			.attr('stroke-width', 4)
			.attr('stroke', '#ff6b6b');
	} else {
		// Left-click: start node dragging 
		isDraggingForEdge = false;
		
		// Store drag start position and node for click detection
		dragStartPosition = { x: event.x, y: event.y };
		draggedNode = d;
		
		// If dragging a selected node and there are multiple selected nodes, prepare for multi-drag
		if (selectedNodeIds.has(d.id) && selectedNodeIds.size > 1) {
			// Calculate offsets for all selected nodes relative to the dragged node
			dragOffsets.clear();
			nodes.forEach(node => {
				if (selectedNodeIds.has(node.id)) {
					dragOffsets.set(node.id, {
						dx: node.x - d.x,
						dy: node.y - d.y
					});
				}
			});
		} else {
			// Single node drag - clear offsets
			dragOffsets.clear();
		}
		
		if (mode === 'force') {
			// Fix positions for all selected nodes if multi-dragging
			if (selectedNodeIds.has(d.id) && selectedNodeIds.size > 1) {
				nodes.forEach(node => {
					if (selectedNodeIds.has(node.id)) {
						node.fx = node.x;
						node.fy = node.y;
					}
				});
			} else {
				d.fx = d.x;
				d.fy = d.y;
			}
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
		// Check if we're doing multi-select drag
		if (selectedNodeIds.has(d.id) && selectedNodeIds.size > 1 && dragOffsets.size > 0) {
			// Multi-node drag: move all selected nodes together
			const deltaX = event.x - d.x;
			const deltaY = event.y - d.y;
			
			if (mode === 'force') {
				// Update positions for all selected nodes
				nodes.forEach(node => {
					if (selectedNodeIds.has(node.id)) {
						const offset = dragOffsets.get(node.id) || { dx: 0, dy: 0 };
						node.fx = event.x + offset.dx;
						node.fy = event.y + offset.dy;
						node.x = event.x + offset.dx;
						node.y = event.y + offset.dy;
					}
				});
			} else {
				// Manual mode: update positions and apply repulsion for all selected nodes
				const nodesToUpdate: Node[] = [];
				nodes.forEach(node => {
					if (selectedNodeIds.has(node.id)) {
						const offset = dragOffsets.get(node.id) || { dx: 0, dy: 0 };
						node.x = event.x + offset.dx;
						node.y = event.y + offset.dy;
						nodesToUpdate.push(node);
					}
				});
				
				// Apply repulsion force for each moved node
				nodesToUpdate.forEach(node => applyRepulsionForce(node));
				
				// Trigger reactivity
				nodes = [...nodes];
			}
		} else {
			// Single node drag
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
		}
	} else if (isDraggingForEdge && dragSourceNode && tempEdgeLine) {
		const [x, y] = [event.x, event.y];
		const dx = x - dragSourceNode.x;
		const dy = y - dragSourceNode.y;
		const distance = Math.sqrt(dx * dx + dy * dy);
		if (distance > 0) {
			const unitX = dx / distance;
			const unitY = dy / distance;
			const startX = dragSourceNode.x + unitX * dragSourceNode.radius;
			const startY = dragSourceNode.y + unitY * dragSourceNode.radius;
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
		// Prevent context menu after edge creation by setting flag
		justCompletedEdgeCreation = true;
		event.sourceEvent.preventDefault();
		event.sourceEvent.stopPropagation();

		const targetElement = document.elementFromPoint(event.sourceEvent.clientX, event.sourceEvent.clientY);
		const targetNode = findNodeFromElement(targetElement);
		if (targetNode && targetNode.id !== dragSourceNode?.id) {
			createEdge(dragSourceNode!, targetNode);
		} else {
			// Not dropped on a node: open dialog to create new edge+node
			const [x, y] = d3.pointer(event.sourceEvent, g.node());
			openCreateEdgeAndNodeDialog(dragSourceNode!, {x: dragSourceNode!.x, y: dragSourceNode!.y}, {x, y});
		}
		if (tempEdgeLine) {
			tempEdgeLine.remove();
			tempEdgeLine = null;
		}
		isDraggingForEdge = false;
		dragSourceNode = null;
		setTimeout(() => {
			justCompletedEdgeCreation = false;
		}, 100);
	} else {
		// Check if this was actually a click (minimal movement)
		if (dragStartPosition && draggedNode) {
			const dx = event.x - dragStartPosition.x;
			const dy = event.y - dragStartPosition.y;
			const distance = Math.sqrt(dx * dx + dy * dy);
			
			if (distance < clickThreshold) {
				// This was a click, not a drag - handle selection
				if (event.sourceEvent.shiftKey) {
					// Shift-click: toggle node in/out of selection
					if (selectedNodeIds.has(d.id)) {
						selectedNodeIds.delete(d.id);
					} else {
						selectedNodeIds.add(d.id);
					}
				} else {
					// Regular click: clear existing selection and select this node
					selectedNodeIds.clear();
					selectedEdgeIds.clear();
					selectedNodeIds.add(d.id);
				}
				
				// Update the selection state
				selectedNodeIds = new Set(selectedNodeIds);
				selectedEdgeIds = new Set(selectedEdgeIds);
				
				// Update color palette selection indicator
				updateColorPaletteSelection();
			}
		}
				// Reset drag detection variables
		dragStartPosition = null;
		draggedNode = null;
		
		// Clear drag offsets after drag ends
		dragOffsets.clear();
		
		if (mode === 'force') {
			// Release fixed positions for all selected nodes if multi-dragging
			if (selectedNodeIds.size > 1) {
				nodes.forEach(node => {
					if (selectedNodeIds.has(node.id)) {
						node.fx = null;
						node.fy = null;
					}
				});
			} else {
				// Single node
				d.fx = null;
				d.fy = null;
			}
			
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
	// Open dialog instead of using prompt()
	pendingEdge = { source, target };
	createEdgeName = ''; // Clear previous input
	showCreateEdgeDialog = true;
}

function closeCreateEdgeDialog() {
	showCreateEdgeDialog = false;
	pendingEdge = null;
	createEdgeName = '';
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
	// Render nodes
	const nodeSelection = g.selectAll<SVGGElement, Node>('.node')
		.data(nodes, d => d.id);

	const nodeEnter = nodeSelection.enter()
		.append('g')
		.attr('class', 'node')
		.attr('transform', d => `translate(${d.x}, ${d.y})`);
	nodeEnter.append('circle')
		.attr('r', d => d.radius)
		.attr('fill', d => d.color)
		.attr('stroke', '#fff')
		.attr('stroke-width', 2);

	nodeEnter.append('text')
		.attr('text-anchor', 'middle')
		.attr('dy', '.35em')
		.attr('fill', 'white')
		.attr('font-size', '12px')
		.attr('font-weight', 'bold')
		.text(d => d.label);

	// Merge enter and update
	const nodeMerge = nodeEnter.merge(nodeSelection);

	// Drag & deletion
	const drag = d3.drag<SVGGElement, Node>()
		.filter(e => e.button === 0 || e.button === 2)
		.on('start', handleDragStart)
		.on('drag', handleDrag)
		.on('end', handleDragEnd);
	nodeMerge.call(drag);	nodeMerge.on('mousedown', (e, d) => {
		if (e.button === 1) {
			e.preventDefault(); e.stopPropagation(); deleteNode(d);
		}
	});
	// Double click: open edit dialog
	nodeMerge.on('dblclick', (e, d) => {
		e.preventDefault();
		e.stopPropagation();
		openEditDialog(d, 'node');
	});// Highlight selected nodes and update attributes
	nodeMerge.select('circle')
		.attr('r', d => d.radius)  // Update radius for renamed nodes
		.attr('fill', d => d.color)
		.attr('stroke', d => selectedNodeIds.has(d.id) ? '#ffb300' : '#fff')
		.attr('stroke-width', d => selectedNodeIds.has(d.id) ? 4 : 2);

	// Update text content for renamed nodes
	nodeMerge.select('text')
		.text(d => d.label);

	// Update positions
	nodeMerge.attr('transform', d => `translate(${d.x}, ${d.y})`);
	nodeSelection.exit().remove();

	// Render edges
	const edgeSelection = g.selectAll<SVGGElement, Edge>('.edge')
		.data(edges, d => d.id);

	const edgeEnter = edgeSelection.enter()
		.append('g')
		.attr('class', 'edge');

	edgeEnter.append('line')
		.attr('stroke', '#666')
		.attr('stroke-width', 2)
		.attr('marker-end', 'url(#arrowhead)');

	edgeEnter.append('rect')
		.attr('fill', 'white')
		.attr('stroke', '#666')
		.attr('stroke-width', 1)
		.attr('rx', 3);

	edgeEnter.append('text')
		.attr('text-anchor', 'middle')
		.attr('dy', '.35em')
		.attr('font-size', '10px')
		.attr('fill', '#333')
		.text(d => d.name);
	const edgeMerge = edgeEnter.merge(edgeSelection);
	edgeMerge.on('mousedown', (e, d) => {
		if (e.button === 1) { e.preventDefault(); e.stopPropagation(); deleteEdge(d); }
	});
	// Single click: select edge
	edgeMerge.on('click', (e, d) => {
		if (e.button === 0) {
			e.preventDefault(); e.stopPropagation(); selectEdge(d, e);
		}
	});
	// Double click: open edit dialog
	edgeMerge.on('dblclick', (e, d) => {
		if (e.button === 0) {
			e.preventDefault();
			e.stopPropagation();
			openEditDialog(d, 'edge');
		}
	});
	// Highlight selected edges
	edgeMerge.select('line')
		.attr('stroke', d => selectedEdgeIds.has(d.id) ? '#ffb300' : '#666')
		.attr('stroke-width', d => selectedEdgeIds.has(d.id) ? 4 : 2);
	
	// Update edge text content for renamed edges
	edgeMerge.select('text')
		.text(d => d.name);

	edgeMerge.each(function(d) {
		const edgeGroup = d3.select(this);
		const sourceNode = typeof d.source === 'string' ? nodes.find(n => n.id === d.source) : d.source;
		const targetNode = typeof d.target === 'string' ? nodes.find(n => n.id === d.target) : d.target;
		if (sourceNode && targetNode && sourceNode.x !== undefined && sourceNode.y !== undefined && targetNode.x !== undefined && targetNode.y !== undefined) {
			const dx = targetNode.x - sourceNode.x;
			const dy = targetNode.y - sourceNode.y;
			const dist = Math.sqrt(dx*dx + dy*dy);			if (dist > 0) {
				const ux = dx / dist, uy = dy / dist;
				const startX = sourceNode.x + ux * sourceNode.radius;
				const startY = sourceNode.y + uy * sourceNode.radius;
				const endX = targetNode.x - ux * targetNode.radius;
				const endY = targetNode.y - uy * targetNode.radius;
				edgeGroup.select('line')
					.attr('x1', startX).attr('y1', startY)
					.attr('x2', endX).attr('y2', endY);
				const midX = (startX + endX)/2, midY = (startY + endY)/2;
				const text = edgeGroup.select('text');
				const bbox = (text.node() as SVGTextElement).getBBox();
				edgeGroup.select('rect')
					.attr('x', midX - bbox.width/2 -2)
					.attr('y', midY - bbox.height/2 -1)
					.attr('width', bbox.width+4)
					.attr('height', bbox.height+2);
				text.attr('x', midX).attr('y', midY);
			}
		}
	});
	edgeSelection.exit().remove();
}

// --- Selection Functions ---

function selectNode(node: Node, event: MouseEvent) {
	if (event.shiftKey) {
		// Multi-select: toggle node in selection
		if (selectedNodeIds.has(node.id)) {
			selectedNodeIds.delete(node.id);
		} else {
			selectedNodeIds.add(node.id);
		}
	} else {
		// Single select: clear existing selection and select this node
		selectedNodeIds.clear();
		selectedEdgeIds.clear();
		selectedNodeIds.add(node.id);
	}
	// Update the selection state
	selectedNodeIds = new Set(selectedNodeIds);
	selectedEdgeIds = new Set(selectedEdgeIds);
	
	// Update color palette selection indicator
	updateColorPaletteSelection();
}

function selectEdge(edge: Edge, event: MouseEvent) {
	if (event.shiftKey) {
		// Multi-select: toggle edge in selection
		if (selectedEdgeIds.has(edge.id)) {
			selectedEdgeIds.delete(edge.id);
		} else {
			selectedEdgeIds.add(edge.id);
		}
	} else {
		// Single select: clear existing selection and select this edge
		selectedNodeIds.clear();
		selectedEdgeIds.clear();
		selectedEdgeIds.add(edge.id);
	}
	// Update the selection state
	selectedNodeIds = new Set(selectedNodeIds);
	selectedEdgeIds = new Set(selectedEdgeIds);
	
	// Update color palette selection indicator
	updateColorPaletteSelection();
}

function clearSelection() {
	selectedNodeIds.clear();
	selectedEdgeIds.clear();
	selectedNodeIds = new Set();
	selectedEdgeIds = new Set();
	
	// Update color palette selection indicator
	updateColorPaletteSelection();
	render();
}

function changeSelectedNodesColor(color: string) {
	if (selectedNodeIds.size === 0) return;
	
	nodes = nodes.map(node => {
		if (selectedNodeIds.has(node.id)) {
			return { ...node, color };
		}
		return node;
	});
	
	// Re-render to show color changes
	render();
}

function getSelectedNodeColor(): string | null {
	if (selectedNodeIds.size === 0) return null;
	
	// If multiple nodes selected, return the color if they all have the same color
	const selectedNodes = nodes.filter(node => selectedNodeIds.has(node.id));
	if (selectedNodes.length === 0) return null;
	
	const firstColor = selectedNodes[0].color;
	const allSameColor = selectedNodes.every(node => node.color === firstColor);
	
	return allSameColor ? firstColor : null;
}

function handleKeyDown(event: KeyboardEvent) {
	// If focus is in an input or textarea, do not interfere
	const tag = (event.target as HTMLElement)?.tagName;
	if (tag === 'INPUT' || tag === 'TEXTAREA' || (event.target as HTMLElement)?.isContentEditable) {
		return;
	}
	// Delete selected nodes and edges
	if (event.key === 'Delete' || event.key === 'Backspace') {
		event.preventDefault();
		
		// Delete selected edges first
		if (selectedEdgeIds.size > 0) {
			edges = edges.filter(edge => !selectedEdgeIds.has(edge.id));
			selectedEdgeIds.clear();
		}
		
		// Delete selected nodes (and their connected edges)
		if (selectedNodeIds.size > 0) {
			const nodeIdsToDelete = Array.from(selectedNodeIds);
			
			// Remove edges connected to the nodes being deleted
			edges = edges.filter(edge => {
				const sourceId = typeof edge.source === 'string' ? edge.source : edge.source.id;
				const targetId = typeof edge.target === 'string' ? edge.target : edge.target.id;
				return !nodeIdsToDelete.includes(sourceId) && !nodeIdsToDelete.includes(targetId);
			});
			
			// Remove the nodes
			nodes = nodes.filter(node => !selectedNodeIds.has(node.id));
			selectedNodeIds.clear();
		}
		
		// Update reactive state
		selectedNodeIds = new Set();
		selectedEdgeIds = new Set();
		
		// Restart simulation if in force mode
		if (mode === 'force') {
			restartSimulation();
		}
		
		// Re-render
		render();
	}
	
	// Escape key clears selection
	if (event.key === 'Escape') {
		clearSelection();
		render();
	}
}

// --- End Selection Functions ---

function openRenameDialog(item: Node | Edge, type: 'node' | 'edge') {
    selectedItem = item;
    renameType = type;
    renameValue = type === 'node' ? (item as Node).label : (item as Edge).name;
    showRenameDialog = true;
}

function closeRenameDialog() {
    showRenameDialog = false;
    selectedItem = null;
    renameType = null;
    renameValue = '';
}

function confirmRename() {
    if (selectedItem && renameType) {
        if (renameType === 'node') {
            const node = selectedItem as Node;
            node.label = renameValue;
            // Recalculate radius based on new label
            node.radius = calculateNodeRadius(renameValue);
            nodes = [...nodes];
            
            // Update collision detection in simulation if in force mode
            if (mode === 'force' && simulation) {
                simulation.force('collision', d3.forceCollide().radius(d => d.radius + 2).strength(0.7));
                restartSimulation();
            }
            
            // Force re-render
            render();
        } else if (renameType === 'edge') {
            (selectedItem as Edge).name = renameValue;
            edges = [...edges];
            // Force re-render for edge changes
            render();
        }
    }
    closeRenameDialog();
}

// --- Dialog state for editing node/edge properties ---
let showEditDialog = false;
let editType: 'node' | 'edge' | null = null;
let editItem: Node | Edge | null = null;
let editLabel = '';
let editProperties: { key: string; value: string }[] = [];

function openEditDialog(item: Node | Edge, type: 'node' | 'edge') {
    editType = type;
    editItem = item;
    editLabel = type === 'node' ? (item as Node).label : (item as Edge).name;
    // Convert properties object to array for editing
    editProperties = Object.entries(item.properties || {}).map(([key, value]) => ({ key, value }));
    showEditDialog = true;
}
function closeEditDialog() {
    showEditDialog = false;
    editType = null;
    editItem = null;
    editLabel = '';
    editProperties = [];
}
function confirmEdit() {
    if (editItem && editType) {
        if (editType === 'node') {
            const node = editItem as Node;
            node.label = editLabel;
            node.radius = calculateNodeRadius(editLabel);
            // Convert array back to object, filter out empty keys
            node.properties = Object.fromEntries(editProperties.filter(p => p.key.trim() !== '').map(p => [p.key, p.value]));
            nodes = [...nodes];
            if (mode === 'force' && simulation) {
                simulation.force('collision', d3.forceCollide().radius(d => d.radius + 2).strength(0.7));
                restartSimulation();
            }
            render();
        } else if (editType === 'edge') {
            const edge = editItem as Edge;
            edge.name = editLabel;
            edge.properties = Object.fromEntries(editProperties.filter(p => p.key.trim() !== '').map(p => [p.key, p.value]));
            edges = [...edges];
            render();
        }
    }
    closeEditDialog();
}
function addPropertyField() {
    editProperties = [...editProperties, { key: '', value: '' }];
}
function removePropertyField(idx: number) {
    editProperties = editProperties.filter((_, i) => i !== idx);
}

// --- UI: Edit Dialog for Node/Edge ---

</script>

<svelte:head>
	<title>Graph Database Editor (Combined)</title>
	<meta name="description" content="Graph database WYSIWYG editor with force/manual modes" />
</svelte:head>

<section>
	<h1>Graph Database Editor (Force/Manual Switch)</h1>	<p>
		Right-click on canvas to add nodes. Drag nodes to move them. Right-click and drag from one node to another to create edges. Click to select nodes/edges (Shift+click for multi-select). Double-click to rename. Press Delete to remove selected items. Middle-click to instantly delete.
	</p>
	
	<!-- File Operations -->
	<div class="file-controls" style="margin-bottom: 1rem; display: flex; gap: 0.5rem; align-items: center; flex-wrap: wrap; justify-content: center;">
		<button on:click={newGraph} style="background: #4285f4; color: white; border: none; padding: 0.4rem 0.8rem; border-radius: 4px; cursor: pointer;">New</button>
		<button on:click={loadFromFile} style="background: #34a853; color: white; border: none; padding: 0.4rem 0.8rem; border-radius: 4px; cursor: pointer;">Load</button>
		<button on:click={saveToFile} style="background: #ea4335; color: white; border: none; padding: 0.4rem 0.8rem; border-radius: 4px; cursor: pointer;">Save</button>
		<button on:click={downloadGraph} style="background: #ff9800; color: white; border: none; padding: 0.4rem 0.8rem; border-radius: 4px; cursor: pointer;">Download</button>
		<button on:click={clearGraph} style="background: #666; color: white; border: none; padding: 0.4rem 0.8rem; border-radius: 4px; cursor: pointer;">Clear</button>
		
		<span style="margin-left: 1rem; font-size: 0.9rem; color: #666;">
			Auto-save: 
			<label style="cursor: pointer;">
				<input type="checkbox" bind:checked={autoSaveEnabled} style="margin-left: 0.3rem;"> 
				Enabled
			</label>
		</span>
		
		{#if lastSaved}
			<span style="margin-left: 1rem, font-size: 0.8rem, color: #999;">
				Last saved: {lastSaved}
			</span>
		{/if}
	</div>
	
	<div style="margin-bottom:1rem;">
		<label><input type="radio" bind:group={mode} value="force" on:change={() => switchMode('force')}> Global repulsion (default)</label>
		<label style="margin-left:2em;"><input type="radio" bind:group={mode} value="manual" on:change={() => switchMode('manual')}> Local Repulsion</label>
	</div>	<div class="canvas-container" bind:this={canvasContainer}>
		<!-- D3 SVG will be inserted here -->
	</div>{#if showRenameDialog}
		<div class="modal-backdrop" on:contextmenu|preventDefault>
			<div class="modal-dialog" on:contextmenu|preventDefault>
				<h2>Rename {renameType === 'node' ? 'Node' : 'Edge'}</h2>
				<input type="text" bind:value={renameValue} autofocus on:keydown={(e) => { if (e.key === 'Enter') confirmRename(); if (e.key === 'Escape') closeRenameDialog(); }} />
				<div class="modal-actions">
					<button on:click={confirmRename}>OK</button>
					<button on:click={closeRenameDialog}>Cancel</button>
				</div>
			</div>
		</div>
	{/if}
	{#if showCreateNodeDialog}
		<div class="modal-backdrop" on:contextmenu|preventDefault>
			<div class="modal-dialog" on:contextmenu|preventDefault>
				<h2>Create New Node</h2>
				<input type="text" bind:value={createNodeName} placeholder="Enter node name" autofocus on:keydown={(e) => { if (e.key === 'Enter') createNodeFromDialog(); if (e.key === 'Escape') closeCreateNodeDialog(); }} />
				<div class="modal-actions">
					<button on:click={createNodeFromDialog}>OK</button>
					<button on:click={closeCreateNodeDialog}>Cancel</button>
				</div>
			</div>
		</div>
	{/if}{#if showCreateEdgeDialog}
		<div class="modal-backdrop" on:contextmenu|preventDefault>
			<div class="modal-dialog" on:contextmenu|preventDefault>
				<h2>Create New Edge</h2>
				<input type="text" bind:value={createEdgeName} placeholder="Enter edge name" autofocus on:keydown={(e) => { if (e.key === 'Enter') createEdgeFromDialog(); if (e.key === 'Escape') closeCreateEdgeDialog(); }} />
				<div class="modal-actions">
					<button on:click={createEdgeFromDialog}>OK</button>
					<button on:click={closeCreateEdgeDialog}>Cancel</button>
				</div>
			</div>
		</div>
	{/if}{#if showCreateEdgeAndNodeDialog}
		<div class="modal-backdrop" on:contextmenu|preventDefault>
			<div class="modal-dialog" on:contextmenu|preventDefault>
				<h2>Create New Node and Edge</h2>
				<label style="display:block;margin-bottom:0.5em;">
					Edge Name:
					<input type="text" bind:value={newEdgeName} placeholder="Enter edge name" autofocus on:keydown={(e) => { if (e.key === 'Enter') confirmCreateEdgeAndNode(); if (e.key === 'Escape') closeCreateEdgeAndNodeDialog(); }} />
				</label>
				<label style="display:block;margin-bottom:0.5em;">
					Node Name:
					<input type="text" bind:value={newNodeName} placeholder="Enter node name" on:keydown={(e) => { if (e.key === 'Enter') confirmCreateEdgeAndNode(); if (e.key === 'Escape') closeCreateEdgeAndNodeDialog(); }} />
				</label>
				<div class="modal-actions">
					<button on:click={confirmCreateEdgeAndNode} disabled={!newEdgeName.trim() || !newNodeName.trim()}>OK</button>
					<button on:click={closeCreateEdgeAndNodeDialog}>Cancel</button>
				</div>
			</div>
		</div>
	{/if}
	{#if showEditDialog}
    <div class="modal-backdrop" on:contextmenu|preventDefault>
        <div class="modal-dialog" on:contextmenu|preventDefault>
            <h2>Edit {editType === 'node' ? 'Node' : 'Edge'}</h2>
            <input type="text" bind:value={editLabel} autofocus placeholder={editType === 'node' ? 'Node label' : 'Edge name'} on:keydown={(e) => { if (e.key === 'Enter') confirmEdit(); if (e.key === 'Escape') closeEditDialog(); }} />
            <div style="margin-top:1em;">
                <h3 style="margin-bottom:0.5em;">Properties</h3>
                {#each editProperties as prop, idx}
                    <div style="display:flex;gap:0.5em;margin-bottom:0.3em;align-items:center;">
                        <input type="text" bind:value={prop.key} placeholder="Property name" style="width:7em;" />
                        <input type="text" bind:value={prop.value} placeholder="Property value" style="width:12em;" />
                        <button on:click={() => removePropertyField(idx)} title="Remove property" style="color:#b00;">✕</button>
                    </div>
                {/each}
                <button on:click={addPropertyField} style="margin-top:0.3em;">Add Property</button>
            </div>
            <div class="modal-actions">
                <button on:click={confirmEdit}>OK</button>
                <button on:click={closeEditDialog}>Cancel</button>
            </div>
        </div>
    </div>
{/if}
</section>

<style>
/* ...existing styles from your editors... */
section {
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
:global(.canvas-container svg) {
	-webkit-user-select: none;
	-moz-user-select: none;
	-ms-user-select: none;
	user-select: none;
	/* Prevent middle mouse button from triggering page scroll */
	-ms-scroll-chaining: none;
	overscroll-behavior: none;
}
:global(.node) {
	cursor: grab;
	user-select: none;
}
:global(.node:active) {
	cursor: grabbing;
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
:global(.color-palette) {
	position: relative;
	z-index: 100;
}
:global(.color-swatch:hover circle) {
	stroke-width: 2 !important;
	stroke: #000 !important;
}
:global(.selection) {
	fill: #ADD8E6;
	stroke: #ADD8E6;
	fill-opacity: 0.3;
	stroke-opacity: 0.7;
	stroke-width: 2;
	stroke-dasharray: 5, 5;
}
.modal-backdrop {
	position: fixed;
	inset: 0;
	background: rgba(0,0,0,0.25);
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 1000;
	-webkit-user-select: none;
	-moz-user-select: none;
	-ms-user-select: none;
	user-select: none;
}
.modal-dialog {
	background: #fff;
	padding: 2rem 1.5rem 1.5rem 1.5rem;
	border-radius: 8px;
	box-shadow: 0 8px 32px rgba(0,0,0,0.18);
	min-width: 300px;
	max-width: 90vw;
	display: flex;
	flex-direction: column;
	align-items: stretch;
}
.modal-dialog h2 {
	margin-bottom: 1rem;
	font-size: 1.2rem;
	color: #333;
}
.modal-dialog input[type="text"] {
	padding: 0.5rem;
	font-size: 1rem;
	border: 1px solid #ccc;
	border-radius: 4px;
	margin-bottom: 1rem;
}
.modal-actions {
	display: flex;
	gap: 1rem;
	justify-content: flex-end;
}
.modal-actions button {
	padding: 0.5rem 1.2rem;
	font-size: 1rem;
	border: none;
	border-radius: 4px;
	background: #4285f4;
	color: #fff;
	cursor: pointer;
	transition: background 0.2s;
}
.modal-actions button:last-child {
	background: #aaa;
}
.modal-actions button:hover {
	background: #1a73e8;
}
.file-controls button:hover {
	opacity: 0.9;
	transform: translateY(-1px);
	box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}
.file-controls button:active {
	transform: translateY(0);
}
</style>

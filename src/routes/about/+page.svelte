<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';

	let chartContainer;

	// Sample data for the force simulation
	const data = {
		nodes: [
			{ id: "SvelteKit", group: 1 },
			{ id: "Svelte", group: 1 },
			{ id: "D3", group: 2 },
			{ id: "JavaScript", group: 3 },
			{ id: "TypeScript", group: 3 },
			{ id: "Vite", group: 4 },
			{ id: "Node.js", group: 4 },
			{ id: "HTML", group: 5 },
			{ id: "CSS", group: 5 },
			{ id: "Force Simulation", group: 2 },
			{ id: "Data Visualization", group: 2 },
			{ id: "Web Development", group: 6 },
			{ id: "Interactive", group: 6 },
			{ id: "Modern Web", group: 6 }
		],
		links: [
			{ source: "SvelteKit", target: "Svelte", value: 5 },
			{ source: "Svelte", target: "JavaScript", value: 3 },
			{ source: "D3", target: "JavaScript", value: 4 },
			{ source: "D3", target: "Force Simulation", value: 5 },
			{ source: "D3", target: "Data Visualization", value: 5 },
			{ source: "SvelteKit", target: "TypeScript", value: 3 },
			{ source: "SvelteKit", target: "Vite", value: 4 },
			{ source: "Vite", target: "Node.js", value: 2 },
			{ source: "Svelte", target: "HTML", value: 3 },
			{ source: "Svelte", target: "CSS", value: 3 },
			{ source: "Web Development", target: "Interactive", value: 4 },
			{ source: "Web Development", target: "Modern Web", value: 4 },
			{ source: "SvelteKit", target: "Web Development", value: 4 },
			{ source: "D3", target: "Interactive", value: 3 },
			{ source: "Force Simulation", target: "Interactive", value: 4 }
		]
	};

	function createChart() {
		// Clear any existing chart
		if (chartContainer) {
			chartContainer.innerHTML = '';
		}

		// Specify the dimensions of the chart.
		const width = 928;
		const height = 680;

		// Specify the color scale.
		const color = d3.scaleOrdinal(d3.schemeCategory10);

		// The force simulation mutates links and nodes, so create a copy
		// so that re-evaluating this cell produces the same result.
		const links = data.links.map(d => ({...d}));
		const nodes = data.nodes.map(d => ({...d}));

		// Create a simulation with several forces.
		const simulation = d3.forceSimulation(nodes)
			.force("link", d3.forceLink(links).id(d => d.id))
			.force("charge", d3.forceManyBody())
			.force("x", d3.forceX())
			.force("y", d3.forceY());

		// Create the SVG container.
		const svg = d3.create("svg")
			.attr("width", width)
			.attr("height", height)
			.attr("viewBox", [-width / 2, -height / 2, width, height])
			.attr("style", "max-width: 100%; height: auto;");

		// Add a line for each link, and a circle for each node.
		const link = svg.append("g")
			.attr("stroke", "#999")
			.attr("stroke-opacity", 0.6)
		.selectAll("line")
		.data(links)
		.join("line")
			.attr("stroke-width", d => Math.sqrt(d.value));

		const node = svg.append("g")
			.attr("stroke", "#fff")
			.attr("stroke-width", 1.5)
		.selectAll("circle")
		.data(nodes)
		.join("circle")
			.attr("r", 5)
			.attr("fill", d => color(d.group));

		node.append("title")
			.text(d => d.id);

		// Add a drag behavior.
		node.call(d3.drag()
			.on("start", dragstarted)
			.on("drag", dragged)
			.on("end", dragended));
		
		// Set the position attributes of links and nodes each time the simulation ticks.
		simulation.on("tick", () => {
			link
				.attr("x1", d => d.source.x)
				.attr("y1", d => d.source.y)
				.attr("x2", d => d.target.x)
				.attr("y2", d => d.target.y);

			node
				.attr("cx", d => d.x)
				.attr("cy", d => d.y);
		});

		// Reheat the simulation when drag starts, and fix the subject position.
		function dragstarted(event) {
			if (!event.active) simulation.alphaTarget(0.3).restart();
			event.subject.fx = event.subject.x;
			event.subject.fy = event.subject.y;
		}

		// Update the subject (dragged node) position during drag.
		function dragged(event) {
			event.subject.fx = event.x;
			event.subject.fy = event.y;
		}

		// Restore the target alpha so the simulation cools after dragging ends.
		// Unfix the subject position now that it's no longer being dragged.
		function dragended(event) {
			if (!event.active) simulation.alphaTarget(0);
			event.subject.fx = null;
			event.subject.fy = null;
		}

		// Append the SVG to the container
		chartContainer.appendChild(svg.node());

		// Return a cleanup function
		return () => simulation.stop();
	}

	let cleanup;

	onMount(() => {
		cleanup = createChart();
		
		return () => {
			if (cleanup) cleanup();
		};
	});
</script>

<svelte:head>
	<title>About</title>
	<meta name="description" content="About this app" />
</svelte:head>

<div class="text-column">
	<h1>About this app</h1>

	<p>
		This is a <a href="https://svelte.dev/docs/kit">SvelteKit</a> app with an interactive D3 force simulation. 
		You can make your own by typing the following into your command line and following the prompts:
	</p>

	<pre>npx sv create</pre>

	<div class="chart-container">
		<h2>Interactive Technology Network</h2>
		<p>Drag the nodes around to see the force simulation in action!</p>
		<div bind:this={chartContainer} class="chart"></div>
	</div>

	<p>
		The simulation above demonstrates the relationships between various web technologies used in this application.
		Each node represents a technology, and the links show how they're connected. The colors group related technologies together.
	</p>

	<p>
		The <a href="/sverdle">Sverdle</a> page illustrates SvelteKit's data loading and form handling. Try
		using it with JavaScript disabled!
	</p>
</div>

<style>
	.chart-container {
		margin: 2rem 0;
		text-align: center;
	}
	
	.chart {
		border: 1px solid #ddd;
		border-radius: 8px;
		margin: 1rem auto;
		background: #fafafa;
	}
	
	.chart-container h2 {
		margin-bottom: 0.5rem;
	}
	
	.chart-container p {
		margin-bottom: 1rem;
		font-style: italic;
		color: #666;
	}
</style>

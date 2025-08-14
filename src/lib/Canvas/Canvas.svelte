<script lang="ts">
	import { Delaunay } from 'd3';
	import { onMount } from 'svelte';
	import ColourSelector from './ColourSelector.svelte';
	import ColourGradientSelector, { type GradientNode } from './ColourGradientSelector.svelte';
	interface Point {
		x: number;
		y: number;
	}

	let canvas: HTMLCanvasElement | undefined = $state();
	let selectedColour = $state('#FFFFFF');
	let cellColour = $state('#00FF00');
	let gradientNodes = $state<GradientNode[]>([
		{ id: '1', color: '#FF0000', position: 0 },
		{ id: '2', color: '#0000FF', position: 1 }
	]);
	let colorMode = $state<'single' | 'gradient'>('gradient');
	let customColoredCells = $state<Map<number, string>>(new Map());
	let selectedCellIndex = $state<number | null>(null);
	let gradientSelectorRef: ColourGradientSelector | undefined = $state();

	// Derived cell colors - doesn't cause infinite loops
	const getCellColor = (index: number, point: Point) => {
		// Check for custom color first
		if (customColoredCells.has(index)) {
			return customColoredCells.get(index)!;
		}

		// Use gradient color if in gradient mode
		if (colorMode === 'gradient' && gradientSelectorRef && canvasWidth > 0) {
			const normalizedX = point.x / canvasWidth;
			return gradientSelectorRef.interpolateColor(normalizedX);
		}

		// Default to cell color
		return cellColour;
	};

	let {
		image = $bindable(),
		mode = $bindable('manual'), // 'manual' | 'image'
		showVoronoi = $bindable(true),
		showDelaunay = $bindable(false),
		stainedGlassEffect = $bindable(false),
		cellCount = $bindable(100),
		...props
	}: {
		image?: HTMLImageElement;
		mode?: 'manual' | 'image';
		showVoronoi?: boolean;
		showDelaunay?: boolean;
		stainedGlassEffect?: boolean;
		cellCount?: number;
	} = $props();

	let ctx = $derived(canvas?.getContext('2d'));
	let points: Point[] = $state([]);
	// Canvas dimensions
	let canvasWidth = $derived(image?.width || 1600);
	let canvasHeight = $derived(image?.height || 1200);

	onMount(() => {
		const dpr = window.devicePixelRatio;
		const displayHeight = window.innerHeight;
		const displayWidth = window.innerWidth;
		console.log('Canvas dimensions:', displayWidth, displayHeight);
		canvasWidth = displayWidth * dpr;
		canvasHeight = displayHeight * dpr;
	});
	$inspect(canvasWidth, canvasHeight);
	let dragPointIndex = $state(-1);
	let isMouseDown = $state(false);

	function drawPoint(point: Point, erase = false) {
		if (!ctx) return;

		ctx.beginPath();
		ctx.arc(point.x, point.y, 5, 0, Math.PI * 2);
		ctx.fillStyle = erase ? 'white' : selectedColour;
		ctx.fill();
		ctx.closePath();
	}

	// Handle canvas click
	function handleCanvasClick(event: MouseEvent) {
		if (!canvas) return;

		const rect = canvas.getBoundingClientRect();
		const x = event.clientX - rect.left;
		const y = event.clientY - rect.top;

		// Scale coordinates if canvas is scaled
		const scaleX = canvasWidth / rect.width;
		const scaleY = canvasHeight / rect.height;
		const scaledX = x * scaleX;
		const scaledY = y * scaleY;

		if (mode === 'manual') {
			// Check if clicking near an existing point for deletion (right click) or selection
			const clickRadius = 10;
			const nearbyPointIndex = points.findIndex(
				(p) => Math.sqrt((p.x - scaledX) ** 2 + (p.y - scaledY) ** 2) < clickRadius
			);

			if (nearbyPointIndex === -1) {
				// Check if clicking on a Voronoi cell for coloring
				if (event.shiftKey && points.length >= 3) {
					const cellIndex = findCellAtPoint(scaledX, scaledY);
					if (cellIndex !== -1) {
						customColoredCells.set(cellIndex, cellColour);
						generateVoronoi();
						return;
					}
				}

				// Left click on empty space - add point
				if (event.button === 0) {
					// Left click
					points = [...points, { x: scaledX, y: scaledY }];
					drawPoint(points[points.length - 1]);
				}
			}
		}
	}

	// Handle right-click for point removal
	function handleRightClick(event: MouseEvent) {
		if (!canvas || mode !== 'manual') return;

		const rect = canvas.getBoundingClientRect();
		const x = event.clientX - rect.left;
		const y = event.clientY - rect.top;

		// Scale coordinates if canvas is scaled
		const scaleX = canvasWidth / rect.width;
		const scaleY = canvasHeight / rect.height;
		const scaledX = x * scaleX;
		const scaledY = y * scaleY;

		// Check if right-clicking near an existing point
		const clickRadius = 10;
		const nearbyPointIndex = points.findIndex(
			(p) => Math.sqrt((p.x - scaledX) ** 2 + (p.y - scaledY) ** 2) < clickRadius
		);

		if (nearbyPointIndex !== -1) {
			// Remove the point
			points = points.filter((_, index) => index !== nearbyPointIndex);
			// Also remove any custom colors for cells that no longer exist
			if (customColoredCells.has(nearbyPointIndex)) {
				customColoredCells.delete(nearbyPointIndex);
			}
			generateVoronoi();
		}
	}

	// Find which Voronoi cell contains the given point
	function findCellAtPoint(x: number, y: number): number {
		if (!ctx || points.length < 3) return -1;

		const delaunay = Delaunay.from(
			points,
			(d) => d.x,
			(d) => d.y
		);
		return delaunay.find(x, y);
	}

	// Handle mouse down for dragging
	function handleMouseDown(event: MouseEvent) {
		if (mode !== 'manual') return;
		console.log('button is ', event.button);
		const rect = canvas?.getBoundingClientRect();
		if (!rect) return;

		const x = event.clientX - rect.left;
		const y = event.clientY - rect.top;
		const scaleX = canvasWidth / rect.width;
		const scaleY = canvasHeight / rect.height;
		const scaledX = x * scaleX;
		const scaledY = y * scaleY;

		const clickRadius = 10;
		dragPointIndex = points.findIndex(
			(p) => Math.sqrt((p.x - scaledX) ** 2 + (p.y - scaledY) ** 2) < clickRadius
		);

		if (dragPointIndex !== -1) {
			isMouseDown = true;
		}
	}

	// Handle mouse move for dragging
	function handleMouseMove(event: MouseEvent) {
		if (!isMouseDown || dragPointIndex === -1 || mode !== 'manual') return;

		const rect = canvas?.getBoundingClientRect();
		if (!rect) return;

		const x = event.clientX - rect.left;
		const y = event.clientY - rect.top;
		const scaleX = canvasWidth / rect.width;
		const scaleY = canvasHeight / rect.height;
		const scaledX = Math.max(0, Math.min(canvasWidth, x * scaleX));
		const scaledY = Math.max(0, Math.min(canvasHeight, y * scaleY));

		points[dragPointIndex] = { x: scaledX, y: scaledY };
	}

	// Handle mouse up
	function handleMouseUp() {
		isMouseDown = false;
		dragPointIndex = -1;
	}

	// Clear all points
	function clearPoints() {
		points = [];
	}
	// Export current state
	function exportCanvas(): string | null {
		return canvas?.toDataURL('image/png') || null;
	}

	function generateVoronoi() {
		if (!ctx || points.length === 0) return;

		// Clear the canvas first
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);

		// Draw background image if in image mode
		if (mode === 'image' && image) {
			ctx.drawImage(image, 0, 0, canvasWidth, canvasHeight);
		}

		// Need at least 3 points for a meaningful Voronoi diagram
		if (points.length < 3) {
			// Just draw the points if we don't have enough for Voronoi
			points.forEach((point) => drawPoint(point));
			return;
		}

		const delaunay = Delaunay.from(
			points,
			(d) => d.x,
			(d) => d.y
		);

		const voronoi = delaunay.voronoi([0, 0, canvasWidth, canvasHeight]);

		// Fill Voronoi cells with color (without modifying state)
		if (stainedGlassEffect || colorMode === 'gradient') {
			for (let i = 0; i < points.length; i++) {
				const point = points[i];
				const cellColor = getCellColor(i, point);

				ctx.beginPath();
				voronoi.renderCell(i, ctx);
				ctx.fillStyle = cellColor;
				ctx.fill();
			}
		}

		// Draw Voronoi diagram
		if (showVoronoi) {
			ctx.beginPath();
			voronoi.render(ctx); // This creates the path
			ctx.strokeStyle = 'black';
			ctx.lineWidth = 1;
			ctx.stroke(); // Actually draw the lines
		}

		// Draw Delaunay triangulation
		if (showDelaunay) {
			ctx.beginPath();
			delaunay.render(ctx); // This creates the path for triangles
			ctx.strokeStyle = 'blue';
			ctx.lineWidth = 0.5;
			ctx.stroke(); // Actually draw the triangles
		}

		// Draw the seed points
		if (mode === 'manual') {
			points.forEach((point) => drawPoint(point));
		}

		// Optional: Draw bounds
		ctx.beginPath();
		voronoi.renderBounds(ctx);
		ctx.strokeStyle = 'gray';
		ctx.lineWidth = 2;
		ctx.stroke();
	}

	$effect(() => {
		console.log('wow', points);
		generateVoronoi();
	});

	// Public methods
	export { clearPoints, exportCanvas };
</script>

<div class="relative">
	<canvas
		bind:this={canvas}
		class="block h-auto w-full max-w-full cursor-crosshair border border-gray-300"
		width={canvasWidth}
		height={canvasHeight}
		onclick={handleCanvasClick}
		onmousedown={handleMouseDown}
		onmousemove={handleMouseMove}
		onmouseup={handleMouseUp}
		oncontextmenu={(e) => {
			e.preventDefault();
			handleRightClick(e);
		}}
		{...props}
	></canvas>

	{#if mode === 'manual'}
		<div class="bg-opacity-90 absolute bottom-2 left-2 rounded bg-white p-2 text-xs">
			<p>Left click: Add point</p>
			<p>Right click: Remove point</p>
			<p>Drag: Move point</p>
			<p>Shift + Click: Color cell (uses Cell Color)</p>
			<p>Points: {points.length}</p>
		</div>
	{/if}
	<div class="bg-opacity-90 absolute right-2 bottom-2 space-y-2 rounded bg-white p-2 text-xs">
		<!-- Vertex Color -->
		<div>
			<label class="mb-1 block text-xs font-semibold text-gray-700">Vertex Color:</label>
			<ColourSelector bind:selectedColour />
		</div>

		<!-- Cell Color for manual coloring -->
		<div>
			<label class="mb-1 block text-xs font-semibold text-gray-700">Cell Color:</label>
			<ColourSelector bind:selectedColour={cellColour} />
		</div>

		<!-- Cell Color Mode -->
		<div>
			<label class="mb-1 block text-xs font-semibold text-gray-700">Cell Color Mode:</label>
			<select
				bind:value={colorMode}
				class="w-full rounded border px-1 py-0.5 text-xs"
				onchange={() => generateVoronoi()}
			>
				<option value="single">Single</option>
				<option value="gradient">Gradient</option>
			</select>
		</div>

		<!-- Cell Color Controls -->
		{#if colorMode === 'gradient'}
			<div>
				<label class="mb-1 block text-xs font-semibold text-gray-700">Cell Gradient:</label>
				<ColourGradientSelector
					bind:this={gradientSelectorRef}
					bind:gradientNodes
					onGradientChange={() => generateVoronoi()}
				/>
			</div>
		{/if}
	</div>
</div>

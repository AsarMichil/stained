<script lang="ts">
	import { Delaunay } from 'd3';
	interface Point {
		x: number;
		y: number;
	}

	interface VoronoiCell {
		points: Point[];
		color: string;
		center: Point;
	}

	let canvas: HTMLCanvasElement | undefined = $state();
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
	let canvasWidth = $derived(image?.width || 800);
	let canvasHeight = $derived(image?.height || 600);
	let dragPointIndex = $state(-1);
	let isMouseDown = $state(false);

	function drawPoint(point: Point, erase = false) {
		if (!ctx) return;

		ctx.beginPath();
		ctx.arc(point.x, point.y, 5, 0, Math.PI * 2);
		ctx.fillStyle = erase ? 'white' : 'red';
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
				// Left click on empty space - add point
				points = [...points, { x: scaledX, y: scaledY }];
				drawPoint(points[points.length - 1]);
			}
		}
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
		class="block h-auto max-w-full cursor-crosshair border border-gray-300"
		width={canvasWidth}
		height={canvasHeight}
		onclick={handleCanvasClick}
		onmousedown={handleMouseDown}
		onmousemove={handleMouseMove}
		onmouseup={handleMouseUp}
		oncontextmenu={(e) => e.preventDefault()}
		{...props}
	></canvas>

	{#if mode === 'manual'}
		<div class="bg-opacity-90 absolute bottom-2 left-2 rounded bg-white p-2 text-xs">
			<p>Left click: Add point</p>
			<p>Right click: Remove point</p>
			<p>Drag: Move point</p>
			<p>Points: {points.length}</p>
		</div>
	{/if}
</div>

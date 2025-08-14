<script lang="ts">
	import { Popover } from 'melt/components';
	import ColourSelector from './ColourSelector.svelte';

	export interface GradientNode {
		id: string;
		color: string;
		position: number; // 0-1 position on gradient
	}

	let {
		gradientNodes = $bindable([
			{ id: '1', color: '#FF0000', position: 0 },
			{ id: '2', color: '#0000FF', position: 1 }
		]),
		onGradientChange = () => {}
	}: {
		gradientNodes: GradientNode[];
		onGradientChange?: (nodes: GradientNode[]) => void;
	} = $props();

	let draggedNodeId = $state<string | null>(null);
	let gradientPreviewRef: HTMLDivElement | undefined = $state();

	// Generate CSS gradient string
	const gradientCss = $derived(() => {
		const sortedNodes = gradientNodes.toSorted((a, b) => a.position - b.position);
		const colorStops = sortedNodes.map((node) => `${node.color} ${node.position * 100}%`);
		return `linear-gradient(to right, ${colorStops.join(', ')})`;
	});

	// Interpolate color at given position (0-1)
	export function interpolateColor(position: number): string {
		const sortedNodes = gradientNodes.toSorted((a, b) => a.position - b.position);

		if (position <= sortedNodes[0].position) return sortedNodes[0].color;
		if (position >= sortedNodes[sortedNodes.length - 1].position)
			return sortedNodes[sortedNodes.length - 1].color;

		// Find the two nodes to interpolate between
		for (let i = 0; i < sortedNodes.length - 1; i++) {
			const node1 = sortedNodes[i];
			const node2 = sortedNodes[i + 1];

			if (position >= node1.position && position <= node2.position) {
				const t = (position - node1.position) / (node2.position - node1.position);
				return interpolateHexColors(node1.color, node2.color, t);
			}
		}

		return sortedNodes[0].color;
	}

	function interpolateHexColors(color1: string, color2: string, t: number): string {
		const r1 = parseInt(color1.slice(1, 3), 16);
		const g1 = parseInt(color1.slice(3, 5), 16);
		const b1 = parseInt(color1.slice(5, 7), 16);

		const r2 = parseInt(color2.slice(1, 3), 16);
		const g2 = parseInt(color2.slice(3, 5), 16);
		const b2 = parseInt(color2.slice(5, 7), 16);

		const r = Math.round(r1 + (r2 - r1) * t);
		const g = Math.round(g1 + (g2 - g1) * t);
		const b = Math.round(b1 + (b2 - b1) * t);

		return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
	}

	function updateNodeColor(nodeId: string, newColor: string) {
		gradientNodes = gradientNodes.map((node) =>
			node.id === nodeId ? { ...node, color: newColor } : node
		);
		onGradientChange(gradientNodes);
	}

	function addNode() {
		const newPosition = 0.5; // Add in the middle by default
		const newId = Date.now().toString();
		const interpolatedColor = interpolateColor(newPosition);

		gradientNodes = [
			...gradientNodes,
			{ id: newId, color: interpolatedColor, position: newPosition }
		];
		onGradientChange(gradientNodes);
	}

	function removeNode(nodeId: string) {
		if (gradientNodes.length <= 2) return; // Keep at least 2 nodes
		gradientNodes = gradientNodes.filter((node) => node.id !== nodeId);
		onGradientChange(gradientNodes);
	}

	function handleNodeMouseDown(event: MouseEvent, nodeId: string) {
		draggedNodeId = nodeId;
		event.preventDefault();
	}

	function handleMouseMove(event: MouseEvent) {
		if (!draggedNodeId || !gradientPreviewRef) return;

		const rect = gradientPreviewRef.getBoundingClientRect();
		const x = event.clientX - rect.left;
		const newPosition = Math.max(0, Math.min(1, x / rect.width));

		gradientNodes = gradientNodes.map((node) =>
			node.id === draggedNodeId ? { ...node, position: newPosition } : node
		);
		onGradientChange(gradientNodes);
	}

	function handleMouseUp() {
		draggedNodeId = null;
	}

	// Add global mouse event listeners
	$effect(() => {
		if (typeof window !== 'undefined') {
			window.addEventListener('mousemove', handleMouseMove);
			window.addEventListener('mouseup', handleMouseUp);

			return () => {
				window.removeEventListener('mousemove', handleMouseMove);
				window.removeEventListener('mouseup', handleMouseUp);
			};
		}
	});
</script>

<Popover>
	{#snippet children(popover)}
		<button
			{...popover.trigger}
			class="flex items-center gap-2 rounded-lg border-2 border-gray-300 px-4 py-3 font-medium shadow-sm transition-all duration-200 hover:-translate-y-0.5 hover:shadow-md"
		>
			<div class="h-6 w-12 rounded border" style="background: {gradientCss};"></div>
			<span class="text-sm">Gradient</span>
		</button>

		<div
			{...popover.content}
			class="z-50 min-w-80 rounded-xl border border-gray-300 bg-white p-5 shadow-xl"
		>
			<div class="mb-4">
				<label class="mb-2 block text-sm font-semibold text-gray-700"> Color Gradient </label>

				<!-- Gradient Preview -->
				<div class="relative">
					<div
						bind:this={gradientPreviewRef}
						class="h-8 w-full cursor-crosshair rounded border"
						style="background: {gradientCss};"
						ondblclick={(e) => {
							const rect = gradientPreviewRef?.getBoundingClientRect();
							if (!rect) return;
							const x = e.clientX - rect.left;
							const position = x / rect.width;
							const newId = Date.now().toString();
							const color = interpolateColor(position);
							gradientNodes = [...gradientNodes, { id: newId, color, position }];
							onGradientChange(gradientNodes);
						}}
					></div>

					<!-- Gradient Nodes -->
					{#each gradientNodes as node (node.id)}
						<div
							class="absolute -top-1 h-6 w-6 -translate-x-3 transform cursor-grab rounded-full border-2 border-white shadow-md transition-transform hover:scale-110"
							style="background-color: {node.color}; left: {node.position * 100}%;"
							onmousedown={(e) => handleNodeMouseDown(e, node.id)}
							ondblclick={() => removeNode(node.id)}
							role="button"
							tabindex="0"
						></div>
					{/each}
				</div>

				<p class="mt-2 text-xs text-gray-500">
					Double-click on gradient to add node, double-click node to remove
				</p>
			</div>

			<!-- Color Controls for Each Node -->
			<div class="mb-4 max-h-40 overflow-y-auto">
				<p class="mb-2 text-sm font-semibold text-gray-700">Node Colors</p>
				{#each gradientNodes.toSorted((a, b) => a.position - b.position) as node (node.id)}
					<div class="mb-2 flex items-center gap-2">
						<div class="h-6 w-6 rounded border" style="background-color: {node.color};"></div>
						<span class="text-xs text-gray-600">{Math.round(node.position * 100)}%</span>
						<div class="flex-1">
							<ColourSelector
								selectedColour={node.color}
								onchange={(newColor) => updateNodeColor(node.id, newColor)}
							/>
						</div>
						{#if gradientNodes.length > 2}
							<button
								class="text-xs text-red-500 hover:text-red-700"
								onclick={() => removeNode(node.id)}
							>
								×
							</button>
						{/if}
					</div>
				{/each}
			</div>

			<div class="flex gap-2">
				<button
					class="flex-1 rounded bg-blue-500 px-3 py-2 text-sm text-white hover:bg-blue-600"
					onclick={addNode}
				>
					Add Node
				</button>
			</div>
		</div>
	{/snippet}
</Popover>

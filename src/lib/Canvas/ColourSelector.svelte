<script lang="ts">
	import { Popover } from 'melt/components';

	let {
		selectedColour = $bindable('#FFFFFF'),
		onchange = () => {}
	}: {
		selectedColour: string;
		onchange?: (color: string) => void;
	} = $props();
	let inputValue = $state('#FFFFFF');

	// Function to validate hex color
	function isValidHex(hex: string): boolean {
		const cleanHex = hex.startsWith('#') ? hex.slice(1) : hex;
		return /^[0-9A-Fa-f]{6}$/.test(cleanHex);
	}

	// Function to normalize hex input (add # if missing)
	function normalizeHex(hex: string): string {
		const cleanHex = hex.startsWith('#') ? hex : `#${hex}`;
		return cleanHex.toUpperCase();
	}

	// Handle input changes
	function handleInputChange(event: Event) {
		const target = event.target as HTMLInputElement;
		const value = target.value;
		inputValue = value;

		if (isValidHex(value)) {
			selectedColour = normalizeHex(value);
			onchange(selectedColour);
		}
	}

	// Calculate contrast color for text
	function getContrastColor(hex: string): string {
		const cleanHex = hex.replace('#', '');
		const r = parseInt(cleanHex.substr(0, 2), 16);
		const g = parseInt(cleanHex.substr(2, 2), 16);
		const b = parseInt(cleanHex.substr(4, 2), 16);

		// Calculate relative luminance
		const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255;

		return luminance > 0.5 ? '#000000' : '#FFFFFF';
	}
</script>

<Popover>
	{#snippet children(popover)}
		<button
			{...popover.trigger}
			class="cursor-pointer rounded-lg border-2 border-gray-300 px-4 py-3 font-mono font-medium shadow-sm transition-all duration-200 hover:-translate-y-0.5 hover:shadow-md"
			style="background-color: {selectedColour}; color: {getContrastColor(selectedColour)};"
		>
			{selectedColour}
		</button>

		<div
			{...popover.content}
			class="z-50 min-w-72 rounded-xl border border-gray-300 bg-white p-5 shadow-xl"
		>
			<div class="mb-4">
				<label for="hex-input" class="mb-2 block text-sm font-semibold text-gray-700">
					Hex Color
				</label>
				<input
					id="hex-input"
					type="text"
					maxLength="7"
					bind:value={inputValue}
					oninput={handleInputChange}
					placeholder="#FFFFFF"
					class="w-full rounded-md border-2 px-3 py-2.5 font-mono text-sm transition-colors duration-200 outline-none {isValidHex(
						inputValue
					)
						? 'border-emerald-500'
						: 'border-red-500'}"
				/>
				{#if !isValidHex(inputValue)}
					<p class="mt-1 text-xs text-red-500">
						Please enter a valid hex color (e.g., #FF5733 or FF5733)
					</p>
				{/if}
			</div>

			<div class="mb-4">
				<p class="mb-2 text-sm font-semibold text-gray-700">Preview</p>
				<div
					class="flex h-15 w-full items-center justify-center rounded-lg border-2 border-gray-300 font-medium"
					style="background-color: {selectedColour}; color: {getContrastColor(selectedColour)};"
				>
					{selectedColour}
				</div>
			</div>

			<div class="flex flex-wrap gap-2">
				<p class="mb-2 w-full text-sm font-semibold text-gray-700">Quick Colors</p>
				{#each ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7', '#DDA0DD', '#98D8C8', '#F7DC6F'] as color (color)}
					<button
						id={`quick-color-${color}`}
						class="h-8 w-8 cursor-pointer rounded-md border-2 transition-all duration-200 hover:scale-110 {selectedColour ===
						color
							? 'border-gray-700'
							: 'border-gray-300'}"
						style="background-color: {color};"
						onclick={() => {
							selectedColour = color;
							inputValue = color;
							onchange(selectedColour);
						}}
					></button>
				{/each}
			</div>
		</div>
	{/snippet}
</Popover>

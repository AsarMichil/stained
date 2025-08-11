<script lang="ts">
	import Canvas from '$lib/Canvas/Canvas.svelte';
	let image: HTMLImageElement | undefined = $state();
</script>

<h1>It should not be hard to make a library that creates stained glass</h1>
<input
	type="file"
	accept="image/*"
	onchange={(e) => {
		console.log(e);
		const file = e.currentTarget.files[0];
		console.log(file);
		if (file) {
			const reader = new FileReader();
			reader.onload = (event) => {
				const img = new Image();
				img.src = event.currentTarget.result as string;
				img.onload = () => {
					image = img;
				};
			};
			reader.readAsDataURL(file);
		}
	}}
/>
<Canvas {image}></Canvas>

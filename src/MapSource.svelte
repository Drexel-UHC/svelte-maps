<script>
	import { getContext, setContext, onMount, onDestroy } from 'svelte';
	
	export let id;
	export let type;
	export let url = null;
	export let props = {};
	export let data = null;
	export let layer = null;
	export let promoteId = null;
	export let minzoom = null;
	export let maxzoom = null;
	export let tilesize = 256;
	
	let loaded = false;
	let urlPrev = url;
	
	const { getMap } = getContext('map');
	const map = getMap();

	setContext("source", {
		source: id,
		layer: layer,
		promoteId: promoteId
	});
	
	if (map.getSource(id)) {
    map.removeSource(id);
	}

	function sleep (ms = 1000) {
		return new Promise((resolve) => setTimeout(resolve, ms));
	}

	async function isSourceLoaded(){
		await sleep(100);

    if (map.isSourceLoaded(id)) {
			loaded = true;
			console.log(id + ' map source loaded!');
    } else {
			console.log('...');
			isSourceLoaded();
		}
	}
	
	// Set optional source properties
	if (minzoom) {
    props.minzoom = minzoom;
	}
	if (maxzoom) {
    props.maxzoom = maxzoom;
	}
	if (layer && promoteId) {
		props.promoteId = {};
		props.promoteId[layer] = promoteId;
	} else if (promoteId) {
		props.promoteId = promoteId;
	}
	
	function addSource() {
		console.log(id + ' map source loading...');
		let layerdef;
		
  	if (type == "geojson") {
	  	if (data) {
		  	layerdef = {
	  		  type,
	  		  data,
					...props
				};
  		} else if (url) {
	  		layerdef = {
	  		  type,
	  		  data: url,
					...props
				};
		  }
	  } else if (type == "vector") {
	  	layerdef = {
	  		type,
	  		...props
			};
			if (url.slice(0, 7) === "pmtiles") {
				layerdef.url = url;
			} else {
				layerdef.tiles = [url];
			}
		} else if (type == "raster") {
	  	layerdef = {
	  		type,
	  		tiles: [ url ],
				tileSize: tilesize,
	  		...props
			};
		} else if (type == "raster-dem") {
			layerdef = {
				type,
				tiles: [ url ],
				tileSize: tilesize,
				...props
			}
		}
		if (layerdef) {
			console.log(layerdef)
			map.addSource(id, layerdef);
			isSourceLoaded();
		}
	};

	function setData(data) {
		let source = map.getSource(id);
		if (source) source.setData(data);
	}
	$: type == "geojson" && loaded && setData(data);

	function setVectorTiles(url) {
		if (url !== urlPrev) {
			let source = map.getSource(id);
			if (source) {
				if (url.slice(0, 7) === "pmtiles") {
					source.setUrl(url);
				} else {
					source.setTiles([url]);
				}
			}
			urlPrev = url;
		}
	}
	$: type == "vector" && loaded && setVectorTiles(url);

	function setRasterTiles(url) {
		if (url !== urlPrev) {
			map.getSource(id).tiles = [ url ];
			map.style.sourceCaches[id].clearTiles();
			map.style.sourceCaches[id].update(map.transform);
			map.triggerRepaint();
			urlPrev = url;
		}
	}
	$: type == "raster" && loaded && setRasterTiles(url);
	
	onMount(addSource);

	onDestroy(async () => {
		if (map && map.getSource(id)) {
			let layers = map.getStyle().layers;
			layers.filter(l => l.source == id)
			.forEach(l => {
				map.removeLayer(l.id);
			});

			map.removeSource(id);
		}
	});
</script>

{#if loaded}
	<slot></slot>
{/if}
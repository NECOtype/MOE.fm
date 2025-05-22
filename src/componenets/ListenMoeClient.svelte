<script>
	import { onMount } from "svelte";

	/** @type {any} **/
	let track = null;

	/** @type {HTMLAudioElement} **/
	let audio;

	/** @type {boolean} **/
	let isPaused = true;

	/** @type {NodeJS.Timeout | undefined } **/
	let heartbeatInterval;

	/** @type {WebSocket | undefined } **/
	let ws;

	/** Sends heartbeat packets every `interval`ms
	 * @param {number} interval
	 **/
	function heartbeat(interval) {
		heartbeatInterval = setInterval(() => {
			ws.send(JSON.stringify({ op: 9 }));
		}, interval);
	}

	/**
	 * Establish WebSocket connection to Listen.moe
	 */
	function connect() {
		ws = new WebSocket("wss://listen.moe/gateway_v2");

		ws.onopen = () => {
			clearInterval(heartbeatInterval);
		};

		ws.onmessage = (message) => {
			if (!message.data.length) return;
			let response;
			try {
				response = JSON.parse(message.data);
			} catch {
				return;
			}

			switch (response.op) {
				case 0:
					ws.send(JSON.stringify({ op: 9 }));
					heartbeat(response.d.heartbeat);
					break;
				case 1:
					if (
						[
							"TRACK_UPDATE",
							"TRACK_UPDATE_REQEST",
							"QUEUE_UPDATE",
							"NOTIFICATION",
						].includes(response.t)
					) {
						track = response.d;
					}
					break;
				default:
					break;
			}
		};

		ws.onclose = () => {
			clearInterval(heartbeatInterval);
			setTimeout(() => connect(), 5000);
		};
	}

	const togglePlayback = () => {
		if (!audio) return;
		if (audio.paused) {
			audio.play();
		} else {
			audio.pause();
		}
	};

	onMount(() => {
		connect();
		return () => {
			clearInterval(heartbeatInterval);
			ws?.close();
		};
	});
</script>

{#if track}
	<div class="player bg-zinc-900 p-4 shadow-md shadow-zinc-300 w-80">
		<img
			class="w-full block shadow-xl shadow-black"
			src={track.song.albums.length === 0
				? `https://cdn.listen.moe/covers/${track.song.albums[0].image}`
				: "https://i.postimg.cc/Ss7pF5XZ/cover.png"}
			alt="album-cover"
		/>
		<div class="p-3">
			<p class="text-xl text-center text-zinc-200">
				{track.song.title}
			</p>
			<p class="text-gray-400 text-sm text-center">
				{track.song.artists.map((a) => a.name).join(", ")}
			</p>
		</div>
		<div class="controls flex items-center justify-center">
			<button
				class="text-gray-400 underline cursor-pointer"
				onclick={togglePlayback}>{isPaused ? "play" : "pause"}</button
			>
		</div>
	</div>
	<audio
		style="display: none"
		bind:this={audio}
		autoplay
		controls
		onloadedmetadata={() => (audio.volume = 0.2)}
	>
		<source src="https://listen.moe/opus" />
	</audio>
{:else}
	<p>Loading current track...</p>
{/if}

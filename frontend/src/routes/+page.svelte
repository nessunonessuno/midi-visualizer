<script>
	import { onMount } from 'svelte';
	import * as Tone from 'tone';
	import * as MidiModule from '@tonejs/midi';

	const Midi = MidiModule.Midi;

	let midiData;
	let playbackState = 'stopped'; // 'playing', 'paused', or 'stopped'
	let synths = [];
	let startTime;
	let isInitialized = false;
	onMount(async () => {
		// Load MIDI file
		const midi = await Midi.fromUrl('megalovania.mid');
		midiData = midi;
	});

	let canvas;
	let canvasContext;
	let analyser;

	function setupVisualization() {
		canvas = document.getElementById('visualization');
		canvasContext = canvas.getContext('2d');
		analyser = Tone.getContext().createAnalyser();
		analyser.fftSize = 2048;
	}

	function drawWaveform() {
		if (!analyser) return;

		const bufferLength = analyser.frequencyBinCount;
		const dataArray = new Uint8Array(bufferLength);

		analyser.getByteTimeDomainData(dataArray);

		canvasContext.fillStyle = 'rgb(200, 200, 200)';
		canvasContext.fillRect(0, 0, canvas.width, canvas.height);

		canvasContext.lineWidth = 2;
		canvasContext.strokeStyle = 'rgb(0, 0, 0)';

		canvasContext.beginPath();

		const sliceWidth = (canvas.width * 1.0) / bufferLength;
		let x = 0;

		for (let i = 0; i < bufferLength; i++) {
			const v = dataArray[i] / 128.0;
			const y = (v * canvas.height) / 2;

			if (i === 0) {
				canvasContext.moveTo(x, y);
			} else {
				canvasContext.lineTo(x, y);
			}

			x += sliceWidth;
		}

		canvasContext.lineTo(canvas.width, canvas.height / 2);
		canvasContext.stroke();

		window.animationFrameId = requestAnimationFrame(drawWaveform);
	}

	function playMidi() {
		if (!midiData || playbackState === 'playing') return;

		// Stop any ongoing playback
		stopMidi();

		// Ensure auio context is started
		Tone.start().then(() => {
			const now = Tone.now();

			if (!isInitialized) {
				setupVisualization();
				synths = midiData.tracks.map(() => new Tone.PolySynth().connect(analyser).toDestination());
				isInitialized = true;
			} else {
				// If already initialized, just reconnect synths
				synths.forEach((synth) => synth.connect(analyser).toDestination());
			}

			// Clear all previous events
			Tone.Transport.cancel();

			midiData.tracks.forEach((track, trackIndex) => {
				const synth = synths[trackIndex];

				track.notes.forEach((note) => {
					Tone.Transport.schedule((time) => {
						synth.triggerAttackRelease(note.name, note.duration, time, note.velocity);
					}, note.time);
				});
			});

			drawWaveform();
			startTime = now;

			Tone.Transport.start();
			playbackState = 'playing';
		});
	}

	function stopMidi() {
		Tone.Transport.stop();
		Tone.Transport.cancel(); // Clear all scheduled events
		if (synths.length > 0) {
			synths.forEach((synth) => {
				synth.releaseAll();
				synth.disconnect();
			});
		}
		playbackState = 'stopped';
		if (analyser) {
			analyser.disconnect();
		}
		if (canvasContext) {
			canvasContext.clearRect(0, 0, canvas.width, canvas.height);
		}
		// Cancel any ongoing animation frame
		if (window.animationFrameId) {
			cancelAnimationFrame(window.animationFrameId);
			window.animationFrameId = null;
		}
	}
</script>

<button on:click={playMidi} disabled={playbackState === 'playing'}>
	{playbackState === 'paused' ? 'Resume' : 'Play'} MIDI
</button>
<button on:click={stopMidi} disabled={playbackState === 'stopped'}>Stop MIDI</button>
<canvas id="visualization" width="800" height="200"></canvas>

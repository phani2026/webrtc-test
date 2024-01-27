<script lang="ts">
	import Counter from './Counter.svelte';
	import welcome from '$lib/images/svelte-welcome.webp';
	import welcome_fallback from '$lib/images/svelte-welcome.png';

	import ClipboardIcon from './icons/ClipboardIcon.svelte';
	import { browser } from '$app/environment';




	let connection: RTCPeerConnection;
	let dataChannel: RTCDataChannel;
	let isConnecting = false;

	let tconnection: RTCPeerConnection;
	let tdataChannel: RTCDataChannel;
	let tisConnecting = false;

	let generating = false;
	let offerLink = '';
	let pastedOfferLink = ''
	let showOfferLink = false;
	let answerSDP = '';
	let pastedAnswer = '';
  	let showAnswerCode = false;



	async function handleInput(event: Event & { target: HTMLInputElement }) {
		pastedOfferLink = event.target.value;
	}


	export function sdpEncode(offer: RTCSessionDescriptionInit): string {
		const encodedOffer: string = JSON.stringify(offer);
		return encodedOffer;
	}

	export function sdpDecode(s: string): RTCSessionDescriptionInit {
		const decodedOffer: RTCSessionDescriptionInit = JSON.parse(s);
		return decodedOffer;
	}

	async function createSDPLink(offer: RTCSessionDescription) {
		console.log(offer.sdp)
		offerLink = sdpEncode(offer);
		console.log(offerLink)
	}

	async function createPeerAndDataCannel() {
		connection = new RTCPeerConnection({
			iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
		});

		connection.onicecandidateerror = () => {
			console.log('ICE Candidate error', 'error');
		};

		const video:any = document.getElementById("sourceVideo");

		navigator.mediaDevices
			.getDisplayMedia({ video: true })
			.then(stream => {
				console.log('streaming track connection',stream.id,stream.getVideoTracks()[0])
				connection.addTrack(stream.getVideoTracks()[0], stream);

				video.srcObject = stream;
				video.onloadedmetadata = () => {
						video.play();
					};
			})
			.catch(error => {
				// Handle errors
			});

		dataChannel = connection.createDataChannel('data', {
			ordered: false // we handle the order by response status
		});
		dataChannel.onopen = () => {
			console.log('Connected', 'success');
			isConnecting = true;
			dataChannel.send("hi from sender")
		};
		dataChannel.onmessage = (event) => {
			console.log('Received data',event.data)
		};
		dataChannel.onerror = () => {
			console.log('WebRTC error', 'error');
			isConnecting = false;
			offerLink = '';
		};
		dataChannel.onclose = () => {
			console.log('Disconnected', 'error');
			isConnecting = false;
			offerLink = '';
		};
  	}

  	async function generateOfferLink() {
    	generating = true;
    	await createPeerAndDataCannel();

		connection.onicecandidate = async (event) => {
			if (!event.candidate && connection.localDescription) {
				await createSDPLink(connection.localDescription);
				generating = false;
			}
		};

		const offer = await connection.createOffer({
			offerToReceiveVideo: true,
			offerToReceiveAudio: true
		});

    	await connection.setLocalDescription(offer);

    	// stop waiting for ice candidates if longer than timeout
		setTimeout(async () => {
			if (!connection.localDescription || !generating) return;
			console.log('timeout waiting ICE candidates');
			await createSDPLink(connection.localDescription);
			generating = false;
		}, 30000);
  	}

	async function copyOfferLink() {
		await navigator.clipboard.writeText(offerLink);
		console.log('Copied to clipboard', 'info');
	}

	async function acceptAnswer() {
		const remoteDesc: RTCSessionDescriptionInit = {
			type: 'answer',
			sdp: sdpDecode(pastedAnswer).sdp
		};
		await connection.setRemoteDescription(remoteDesc);

		// navigator.mediaDevices.getUserMedia({ video: true, audio: false })
		// 	.then(stream => {
		// 		console.log('starting stream')
		// 		// Access tracks within the stream
		// 		stream.getTracks().forEach(track => connection.addTrack(track, stream))

		// 		const video:any = document.getElementById("sourceVideo");
		// 				video.srcObject = stream;
		// 				video.onloadedmetadata = () => {
		// 				video.play();
		// 			};

		// 	})
		// 	.catch(error => {
		// 		// Handle errors
		// 		console.log('error stream')
		// 	});

	}



	async function createReceiverPeerAndDataCannel() {
		tconnection = new RTCPeerConnection({
			iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
		});

		tconnection.ondatachannel = (event) => {
			tdataChannel = event.channel;

			tdataChannel.onopen = () => {
				console.log('Connected', 'success');
				tisConnecting = true;
			};
			tdataChannel.onmessage = (event) => {
				console.log('on data',event.data);
			};
			tdataChannel.onerror = () => {
				console.log('WebRTC error', 'error');
				tisConnecting = false;
			};
			tdataChannel.onclose = () => {
				console.log('Disconnected', 'error');
				tisConnecting = false;
			};
		};

		const videoElement:any = document.getElementById("remoteVideo"); // Assuming a video element with ID "remoteVideo"

		tconnection.ontrack = event => videoElement.srcObject = new MediaStream([event.track]);


		// tconnection.ontrack = (event) =>{
		// 	console.log('on track')
		// 	let ttrack:MediaStreamTrack = event.track;

		// 	ttrack.onended = () => {
		// 		console.log('track ended');
		// 	};

		// 	if (ttrack.kind === "video") {
		// 		videoElement.srcObject = new MediaStream([ttrack]);
		// 		videoElement.play(); // Start playing the video
		// 	}
		// }

	}

	async function setOfferSDP(pastedOfferLink: string) {
		console.log('pastedOfferLink',pastedOfferLink)
		const sdpDecoded = sdpDecode(pastedOfferLink);
		console.log('sdpDecoded',sdpDecoded)
		const sessionDesc: RTCSessionDescriptionInit = {
			type: 'offer',
			sdp: sdpDecoded.sdp
		};
    	await tconnection.setRemoteDescription(sessionDesc);
  	}
	
  	

	async function copyAnswerCode() {
		await navigator.clipboard.writeText(answerSDP);
		console.log('Copied to clipboard', 'info');
	}

  	async function generateAnswerSDP() {

		tconnection.onicecandidate = (event) => {
			if (!event.candidate && tconnection.localDescription) {
				answerSDP = sdpEncode(tconnection.localDescription);
			}
		};

		const answer = await tconnection.createAnswer();
		await tconnection.setLocalDescription(answer);

		// stop waiting for ice candidates if longer than timeout
		setTimeout(async () => {
		if (!tconnection.localDescription || answerSDP) return;
			console.log('Timeout waiting ICE candidates');
			answerSDP = sdpEncode(tconnection.localDescription);
		}, 30000);
  	}

	// setOfferSDP(sdpEncoded);
  	// generateAnswerSDP();

	  async function createReceiver() {

		await createReceiverPeerAndDataCannel();
		await setOfferSDP(pastedOfferLink);
		await generateAnswerSDP()

	  }



</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Svelte demo app" />
</svelte:head>

<section>

	<div class="container">
		<div class="col-1">
			<p>Sender</p>
			<br/>
			<button on:click={generateOfferLink}>Generate Offer</button>
			<br/>
			<input
				type='text'
				class="input input-bordered w-full"
				value={offerLink}
				readonly
			/>
			<br/>
			<button on:click={copyOfferLink}>Copy Link</button>
			<br/>
			<input
				type='text'
				class="input input-bordered w-full"
				bind:value={pastedAnswer}
			/>
			<br/>
			<button on:click={acceptAnswer}>Accept answer</button>
			<br/>
			<video id="sourceVideo" ></video>

		</div>
		<div class="col-2">
			<p>Receiver</p>
			<br/>
			<input
					type='text'
					class="input input-bordered w-full"
					bind:value={pastedOfferLink}
				/>
				<br/>
				<button on:click={createReceiver}>Generate Answer</button>
				<br/>
				<input type='text' class="input input-bordered w-full" value={answerSDP}/>
				<br/>
				<button on:click={copyAnswerCode}>Copy Answer</button>
				<br/>
				<video id="remoteVideo" ></video>
		</div>
	  </div>
	






	

	<Counter />
</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: left;
		align-items: left;
	}

	.container {
		display: flex;
	}

	.col-1, .col-2 {
		flex: 1;
	}

	button {
		border-radius: 4px; padding: 5px 5px; font-weight: bold;
		background-color: #007bff;
	}

</style>

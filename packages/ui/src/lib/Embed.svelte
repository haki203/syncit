<script lang="ts">
	import { record } from 'rrweb';
	import type { eventWithTime } from '@rrweb/types';
	import { onMount, onDestroy } from 'svelte';
	import { quintOut } from 'svelte/easing';
	import { scale } from 'svelte/transition';
	import {
		TransporterEvents,
		type TransportAckRecordEvent,
		type Transporter,
		type TransportRemoteControlEvent
	} from '@syncit/core';
	import {
		applyMirrorAction,
		SourceBuffer,
		createEmbedService,
		createEmbedControlService,
		RemoteControlActions,
		CustomEventTags,
		type MirrorPayload
	} from '@syncit/core';
	import { customAlphabet } from 'nanoid';
	import copy from 'copy-to-clipboard';
	import Panel from './components/Panel.svelte';
	import Canvas from './components/Canvas.svelte';
	import PDF from './components/PDF.svelte';
	import Tag from './components/Tag.svelte';
	import Icon from './components/Icon.svelte';
	import { t, setCurrentLanguage } from '../locales';
	import type { PaintingConfig } from './types';

	export let createTransporter: ({ role, uid }: { role: string; uid: string }) => Transporter;
	export let lang: string | undefined = undefined;

	const nanoid = customAlphabet('1234567890abcdef', 10);
	let uid = nanoid();
	const transporter = createTransporter({
		uid,
		role: 'embed'
	});

	let login = transporter.login();
	let ref: HTMLElement;
	$: ref && document.body.appendChild(ref);

	let open = false;

	const buffer = new SourceBuffer<eventWithTime>({
		onTimeout(record) {
			transporter.sendRecord(record);
		}
	});

	let selecting = false;
	let mask: HTMLElement;
	$: mask && document.body.appendChild(mask);
	let blockElSet = new Set<HTMLElement>();
	$: blockEls = Array.from(blockElSet);

	const service = createEmbedService({
		transporter,
		record,
		stopRecordFn: null,
		buffer
	});
	let current = service.state;
	const controlService = createEmbedControlService({
		record
	});
	let controlCurrent = controlService.state;

	const highlight = (target: HTMLElement) => {
		const { x, y, width, height } = target.getBoundingClientRect();
		Object.assign(mask.style, {
			left: `${x}px`,
			top: `${y}px`,
			width: `${width}px`,
			height: `${height}px`,
			display: 'inherit'
		});
	};
	const removeHighlight = () => {
		Object.assign(mask.style, {
			left: 0,
			top: 0,
			width: 0,
			height: 0,
			display: 'none'
		});
	};

	const over = (event: MouseEvent) => {
		if (
			event.target &&
			ref !== event.target &&
			!ref.contains(event.target as HTMLElement) &&
			event.target !== document.body
		) {
			highlight(event.target as HTMLElement);
		}
	};
	const click = (event: MouseEvent) => {
		if (!selecting || ref.contains(event.target as HTMLElement)) {
			return;
		}
		(event.target as HTMLElement).classList.add('rr-block');
		blockElSet = blockElSet.add(event.target as HTMLElement);
		cancelSelect();
	};
	const cancelSelect = () => {
		selecting = false;
		removeHighlight();
		window.removeEventListener('mousemove', over, { capture: true });
		window.removeEventListener('click', click, { capture: true });
	};
	const removeBlockEl = (el: HTMLElement) => {
		blockElSet.delete(el);
		blockElSet = blockElSet;
		removeHighlight();
		el.classList.remove('rr-block');
	};
	function handleSelectBlock() {
		if (selecting) {
			cancelSelect();
		} else {
			selecting = true;
			window.addEventListener('mousemove', over, { capture: true });
			window.addEventListener('click', click, { capture: true });
		}
	}

	function changeMouseSize(level: number) {
		record.addCustomEvent(CustomEventTags.MouseSize, { level });
	}

	function stopRemoteControl() {
		record.addCustomEvent(CustomEventTags.StopRemoteControl, undefined);
		controlService.send('STOP');
	}

	let copied = false;
	function copyUid() {
		copy(uid);
		copied = true;
		setTimeout(() => {
			copied = false;
		}, 1000);
	}

	let painting = false;
	let paintingConfig: PaintingConfig = {
		stroke: '#df4b26',
		strokeWidth: 5,
		mode: 'brush'
	};
	function togglePaint() {
		painting = !painting;
		if (painting) {
			record.addCustomEvent(CustomEventTags.StartPaint, undefined);
			record.addCustomEvent(CustomEventTags.SetPaintingConfig, {
				config: paintingConfig
			});
		} else {
			record.addCustomEvent(CustomEventTags.EndPaint, undefined);
		}
	}
	function setPaintingConfig<T extends keyof typeof paintingConfig>(
		key: T,
		value: (typeof paintingConfig)[T]
	) {
		paintingConfig[key] = value;
		record.addCustomEvent(CustomEventTags.SetPaintingConfig, {
			config: paintingConfig
		});
	}

	let sharingPDF = false;
	let pdfEl: PDF;
	function handlePdf(files: FileList) {
		const file = files[0];
		sharingPDF = true;
		const fileReader = new FileReader();
		fileReader.onload = function () {
			pdfEl.renderPDF({ dataURI: this.result });

			record.addCustomEvent(CustomEventTags.OpenPDF, { dataURI: this.result });
		};
		fileReader.readAsDataURL(file);
	}
	function closePDF() {
		sharingPDF = false;

		record.addCustomEvent(CustomEventTags.ClosePDF, undefined);
	}

	onMount(() => {
		if (lang) {
			setCurrentLanguage(lang);
		}

		service.start();
		service.subscribe((state) => {
			current = state;
		});
		controlService.start();
		controlService.subscribe((state) => {
			controlCurrent = state;
		});

		transporter.on(TransporterEvents.MirrorReady, () => {
			transporter.sendSourceReady();
		});
		transporter.on(TransporterEvents.Start, () => {
			service.send('CONNECT');
		});
		transporter.on(TransporterEvents.AckRecord, ({ payload }) => {
			buffer.delete(payload as TransportAckRecordEvent['payload']);
		});
		transporter.on(TransporterEvents.RemoteControl, ({ payload }) => {
			switch ((payload as TransportRemoteControlEvent['payload']).action) {
				case RemoteControlActions.Request:
					controlService.send('REQUEST');
					break;
				case RemoteControlActions.Stop:
					controlService.send('STOP');
					break;
				default:
					applyMirrorAction(record.mirror, payload as MirrorPayload);
					break;
			}
		});
	});
	onDestroy(() => {
		service.stop();
		controlService.stop();
	});
</script>

<div class="syncit-embed" bind:this={ref}>
	{#if open}
		<div
			transition:scale={{ duration: 500, opacity: 0.5, easing: quintOut }}
			style="transform-origin: right bottom;"
		>
			<Panel>
				{#await login}
					<div class="syncit-center syncit-load-text">
						{t('embed.initializing')}...
					</div>
				{:then}
					<!-- -->
					{#if current.matches('idle')}
						<div class="syncit-center">
							<div class="syncit-panel-control">
								<button class="syncit-btn ordinary" on:click={handleSelectBlock}>
									{selecting ? t('embed.cancel') : t('embed.selectBlockArea')}
								</button>
								<div class="syncit-block-els">
									{#each blockEls as el, idx}
										<Tag
											on:mouseover={() => highlight(el)}
											on:mouseout={removeHighlight}
											on:click={() => removeBlockEl(el)}
										>
											{t('embed.area')}-{idx + 1}
										</Tag>
									{/each}
								</div>
							</div>
							<button class="syncit-btn" on:click={() => service.send('START')}>
								{t('embed.enableShare')}
							</button>
						</div>
					{:else if current.matches('ready')}
						<div class="syncit-center syncit-load-text">
							<div class="syncit-panel-meta">
								{#if copied}
									{t('embed.copied')}
									<!---->
								{:else}
									UID:
									{uid}
									<span class="syncit-copy" on:click={copyUid} on:keypress={copyUid}>
										<Icon name="copy" />
									</span>
								{/if}
							</div>
							<div class="syncit-load-text">{t('embed.ready')}</div>
						</div>
					{:else if current.matches('connected')}
						<div class="syncit-center">
							<div class="syncit-panel-control">
								<button class="syncit-btn ordinary" on:click={togglePaint}
									>{painting ? t('embed.stopPaint') : t('embed.paint')}</button
								>
								{#if painting}
									<div class="syncit-painting-config">
										<div>
											<label for="Stroke">Stroke:</label>
											{paintingConfig.stroke}
											<input
												id="Stroke"
												type="color"
												value={paintingConfig.stroke}
												on:change={(event) =>
													setPaintingConfig('stroke', event.currentTarget.value)}
											/>
										</div>
										<div>
											<label for="strokeWidth">Stroke Width:</label>
											{paintingConfig.strokeWidth}
											<input
												id="strokeWidth"
												type="range"
												min="3"
												max="20"
												value={paintingConfig.strokeWidth}
												on:change={(event) =>
													setPaintingConfig('strokeWidth', Number(event.currentTarget.value))}
											/>
										</div>
										<div>
											<label for="mode">Mode:</label>
											{paintingConfig.mode}
											<!-- svelte-ignore a11y-no-onchange -->
											<select
												id="mode"
												value={paintingConfig.mode}
												on:change={(event) => setPaintingConfig('mode', event.currentTarget.value)}
											>
												<option>brush</option>
												<option>eraser</option>
												<option>highlight</option>
											</select>
										</div>
									</div>
								{/if}
								<input
									type="file"
									on:change={(event) =>
										event.currentTarget.files && handlePdf(event.currentTarget.files)}
									style="overflow: hidden;display:block;width:100%;"
								/>
								<p>{t('embed.mouseSize')}</p>
								<div class="syncit-mouses">
									{#each [t('embed.small'), t('embed.medium'), t('embed.large')] as size, idx}
										<button
											class="{`syncit-mouse-${idx + 1}`} syncit-btn ordinary"
											on:click={() => changeMouseSize(idx + 1)}
										>
											{size}
										</button>
									{/each}
								</div>
								<p>
									{t('embed.remoteControl')}
									<!---->
									{#if controlCurrent.matches('not_control')}
										{t('embed.disabled')}
										<!---->
									{:else if controlCurrent.matches('requesting')}
										{t('embed.requesting')}
										<!---->
									{:else if controlCurrent.matches('controlled')}
										{t('embed.enabled')}
										<!---->
									{/if}
								</p>
								{#if controlCurrent.matches('requesting')}
									<button
										class="syncit-btn ordinary"
										on:click={() => controlService.send('ACCEPT')}
									>
										{t('embed.accept')}
									</button>
								{:else if controlCurrent.matches('controlled')}
									<button class="syncit-btn ordinary" on:click={stopRemoteControl}>
										{t('embed.abortControl')}
									</button>
								{/if}
							</div>
							<button class="syncit-btn" on:click={() => service.send('STOP')}>
								{t('embed.abort')}
							</button>
						</div>
					{/if}
					<!---->
				{:catch error}
					<div class="syncit-center syncit-error">{error.message}</div>
				{/await}
			</Panel>
		</div>
	{/if}
	<!---->
	<button class="syncit-toggle syncit-btn" on:click={() => (open = !open)}>
		<Icon name={open ? 'close' : 'team'} />
	</button>
	<!---->
	<div bind:this={mask} class="syncit-mask" />
</div>

{#if painting}
	<Canvas {...paintingConfig} />
{/if}

{#if sharingPDF}
	<PDF bind:this={pdfEl}>
		<button class="syncit-toggle syncit-btn" on:click={closePDF}>
			<Icon name="close" />
		</button>
	</PDF>
{/if}

<style>
	button {
		outline: none;
	}

	.syncit-embed {
		position: fixed;
		right: 1em;
		bottom: 1em;
		display: flex;
		flex-direction: column;
		z-index: 9999;
	}

	.syncit-btn:hover {
		background: #3399ff;
	}

	.syncit-btn,
	.syncit-btn:active {
		cursor: pointer;
		background: #0078f0;
		border: 1px solid rgba(62, 70, 82, 0.18);
		box-shadow: 0px 1px 2px rgba(184, 192, 204, 0.6);
		color: #fff;
		padding: 8px 16px;
		border-radius: 4px;
		font-size: 14px;
		line-height: 22px;
		margin-bottom: 0.5em;
	}

	.syncit-btn.ordinary {
		background: #fff;
		color: #3e4652;
		border: 1px solid rgba(129, 138, 153, 0.6);
	}
	.syncit-btn.ordinary:hover {
		background: #f5f7fa;
	}
	.syncit-btn.ordinary:active {
		background: #dfe4eb;
	}

	.syncit-toggle,
	.syncit-toggle:active {
		width: 40px;
		height: 40px;
		line-height: 40px;
		border-radius: 20px;
		padding: 0;
		align-self: flex-end;
	}

	.syncit-center {
		width: 100%;
		height: 100%;
		display: flex;
		align-items: center;
		justify-content: center;
		flex-direction: column;
	}

	.syncit-painting-config {
		border-bottom: 1px solid rgba(235, 239, 245, 0.8);
		padding-bottom: 5px;
		margin-bottom: 5px;
		display: flex;
		flex-direction: column;
		align-items: flex-start;
	}

	.syncit-load-text {
		font-size: 14px;
		line-height: 22px;
		color: #3e4652;
	}

	.syncit-error {
		color: #e75a3a;
	}

	.syncit-mask {
		position: fixed;
		left: 0;
		top: 0;
		pointer-events: none;
		z-index: 999999;
		background: rgba(136, 194, 232, 0.75);
		border: 1px solid #3399ff;
		display: none;
	}

	.syncit-panel-control {
		display: flex;
		flex-direction: column;
		flex: 1;
		width: 100%;
		align-items: flex-start;
		border-bottom: 1px solid rgba(235, 239, 245, 0.8);
		margin-bottom: 8px;
	}

	.syncit-panel-meta {
		padding-bottom: 4px;
		margin-bottom: 4px;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.syncit-copy {
		cursor: pointer;
		display: inline-block;
		width: 16px;
		height: 16px;
		margin-left: 4px;
	}

	.syncit-copy:hover {
		opacity: 0.8;
	}

	.syncit-block-els {
		display: flex;
		overflow: auto;
		word-break: keep-all;
	}

	.syncit-block-els > :global(span) {
		cursor: pointer;
		margin-right: 4px;
	}

	.syncit-mouses {
		display: flex;
		justify-content: space-around;
		align-items: center;
	}
</style>

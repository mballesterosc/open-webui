<script lang="ts">
	import DOMPurify from 'dompurify';

	import { onMount, getContext, createEventDispatcher, onDestroy, tick } from 'svelte';
	import * as FocusTrap from 'focus-trap';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import { fade } from 'svelte/transition';
	import { flyAndScale } from '$lib/utils/transitions';

	export let title = '';
	export let htmlContent = '';

	export let cancelLabel = $i18n.t('Cancel');
	export let confirmLabel = $i18n.t('Confirm');

	export let onConfirm = () => {};

	export let show = false;

	$: if (show) {
		init();
	}

	let modalElement = null;
	let contentElement = null;
	let mounted = false;

	let focusTrap: FocusTrap.FocusTrap | null = null;

	const init = () => {
		// Reset any previous state when modal opens
	};

	const handleKeyDown = (event: KeyboardEvent) => {
		if (event.key === 'Escape') {
			console.log('Escape');
			show = false;
			dispatch('cancel');
		}

		if (event.key === 'Enter') {
			// Only submit if Enter is pressed and not in a textarea
			if (!(event.target instanceof HTMLTextAreaElement)) {
				console.log('Enter');
				confirmHandler();
			}
		}
	};

	const serializeForm = (formElement: HTMLElement): Record<string, any> => {
		const formData: Record<string, any> = {};
		
		// Handle text inputs, textareas, selects
		const textInputs = formElement.querySelectorAll('input[type="text"], input[type="email"], input[type="password"], input[type="number"], input[type="tel"], input[type="url"], input[type="search"], input[type="date"], input[type="time"], input[type="datetime-local"], input[type="month"], input[type="week"], input[type="color"], textarea, select');
		textInputs.forEach((input: Element) => {
			const element = input as HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement;
			if (element.name) {
				formData[element.name] = element.value;
			}
		});
		
		// Handle checkboxes
		const checkboxes = formElement.querySelectorAll('input[type="checkbox"]');
		checkboxes.forEach((checkbox: Element) => {
			const element = checkbox as HTMLInputElement;
			if (element.name) {
				if (element.checked) {
					// For multiple checkboxes with same name, create array
					if (element.name in formData) {
						if (Array.isArray(formData[element.name])) {
							formData[element.name].push(element.value || 'on');
						} else {
							formData[element.name] = [formData[element.name], element.value || 'on'];
						}
					} else {
						formData[element.name] = element.value || 'on';
					}
				}
			}
		});
		
		// Handle radio buttons
		const radioButtons = formElement.querySelectorAll('input[type="radio"]:checked');
		radioButtons.forEach((radio: Element) => {
			const element = radio as HTMLInputElement;
			if (element.name) {
				formData[element.name] = element.value;
			}
		});

		// Handle hidden inputs
		const hiddenInputs = formElement.querySelectorAll('input[type="hidden"]');
		hiddenInputs.forEach((input: Element) => {
			const element = input as HTMLInputElement;
			if (element.name) {
				formData[element.name] = element.value;
			}
		});
		console.log(formData)
		return formData;
	};

	const confirmHandler = async () => {		
		await tick();
		await onConfirm();
		
		// Serialize form data from content element
		let formData = {};
        console.log('contentElement:', contentElement);
		if (contentElement) {
			formData = serializeForm(contentElement);
		}
		show = false;
		console.log('Form Data:', formData);
		dispatch('confirm', formData);
	};

	onMount(() => {
		mounted = true;
	});

	$: if (mounted) {
		if (show && modalElement) {
			document.body.appendChild(modalElement);
			focusTrap = FocusTrap.createFocusTrap(modalElement);
			focusTrap.activate();

			window.addEventListener('keydown', handleKeyDown);
			document.body.style.overflow = 'hidden';
		} else if (modalElement) {
			if (focusTrap) {
				focusTrap.deactivate();
			}

			window.removeEventListener('keydown', handleKeyDown);
			document.body.removeChild(modalElement);

			document.body.style.overflow = 'unset';
		}
	}

	onDestroy(() => {
		show = false;
		if (focusTrap) {
			focusTrap.deactivate();
		}
		if (modalElement) {
			document.body.removeChild(modalElement);
		}
	});
</script>

{#if show}
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div
		bind:this={modalElement}
		class=" fixed top-0 right-0 left-0 bottom-0 bg-black/60 w-full h-screen max-h-[100dvh] flex justify-center z-99999999 overflow-hidden overscroll-contain"
		in:fade={{ duration: 10 }}
		on:mousedown={() => {
			show = false;
			dispatch('cancel');
		}}
	>
		<div
			class=" m-auto rounded-2xl max-w-full w-[32rem] mx-2 bg-gray-50 dark:bg-gray-950 max-h-[100dvh] shadow-3xl"
			in:flyAndScale
			on:mousedown={(e) => {
				e.stopPropagation();
			}}
		>
			<div class="px-[1.75rem] py-6 flex flex-col">
				<div class=" text-lg font-semibold dark:text-gray-200 mb-2.5">
					{#if title !== ''}
						{title}
					{:else}
						{$i18n.t('Confirm your action')}
					{/if}
				</div>

				<div class=" text-sm text-gray-500 dark:text-gray-400 flex-1 mb-4">
					{#if htmlContent !== ''}
						<div bind:this={contentElement} class="form-content">
							{@html DOMPurify.sanitize(htmlContent)}
						</div>
					{:else}
						{$i18n.t('Please fill out the form below.')}
					{/if}
				</div>

				<div class="mt-6 flex justify-between gap-1.5">
					<button
						class="bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-white font-medium w-full py-2.5 rounded-lg transition"
						on:click={() => {
							show = false;
							dispatch('cancel');
						}}
						type="button"
					>
						{cancelLabel}
					</button>
					<button
						class="bg-gray-900 hover:bg-gray-850 text-gray-100 dark:bg-gray-100 dark:hover:bg-white dark:text-gray-800 font-medium w-full py-2.5 rounded-lg transition"
						on:click={() => {
							confirmHandler();
						}}
						type="button"
					>
						{confirmLabel}
					</button>
				</div>
			</div>
		</div>
	</div>
{/if}

<style>
	.modal-content {
		animation: scaleUp 0.1s ease-out forwards;
	}

	@keyframes scaleUp {
		from {
			transform: scale(0.985);
			opacity: 0;
		}
		to {
			transform: scale(1);
			opacity: 1;
		}
	}

	/* Form styling for better user experience */
	:global(.form-content) {
		line-height: 1.5;
	}

	:global(.form-content form) {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	:global(.form-content input[type="text"]),
	:global(.form-content input[type="email"]),
	:global(.form-content input[type="password"]),
	:global(.form-content input[type="number"]),
	:global(.form-content input[type="tel"]),
	:global(.form-content input[type="url"]),
	:global(.form-content input[type="search"]),
	:global(.form-content input[type="date"]),
	:global(.form-content input[type="time"]),
	:global(.form-content input[type="datetime-local"]),
	:global(.form-content input[type="month"]),
	:global(.form-content input[type="week"]),
	:global(.form-content input[type="color"]),
	:global(.form-content textarea),
	:global(.form-content select) {
		width: 100%;
		padding: 0.5rem;
		border: 1px solid #d1d5db;
		border-radius: 0.375rem;
		background-color: white;
		color: #374151;
		transition: border-color 0.15s ease-in-out;
	}

	:global(.form-content input[type="text"]:focus),
	:global(.form-content input[type="email"]:focus),
	:global(.form-content input[type="password"]:focus),
	:global(.form-content input[type="number"]:focus),
	:global(.form-content input[type="tel"]:focus),
	:global(.form-content input[type="url"]:focus),
	:global(.form-content input[type="search"]:focus),
	:global(.form-content input[type="date"]:focus),
	:global(.form-content input[type="time"]:focus),
	:global(.form-content input[type="datetime-local"]:focus),
	:global(.form-content input[type="month"]:focus),
	:global(.form-content input[type="week"]:focus),
	:global(.form-content input[type="color"]:focus),
	:global(.form-content textarea:focus),
	:global(.form-content select:focus) {
		outline: none;
		border-color: #3b82f6;
		box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
	}

	:global(.form-content label) {
		display: block;
		margin-bottom: 0.25rem;
		font-weight: 500;
		color: #374151;
	}

	:global(.form-content .checkbox-group),
	:global(.form-content .radio-group) {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	:global(.form-content input[type="checkbox"]),
	:global(.form-content input[type="radio"]) {
		margin-right: 0.5rem;
	}

	:global(.form-content .checkbox-item),
	:global(.form-content .radio-item) {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	/* Dark mode support */
	:global(.dark .form-content input[type="text"]),
	:global(.dark .form-content input[type="email"]),
	:global(.dark .form-content input[type="password"]),
	:global(.dark .form-content input[type="number"]),
	:global(.dark .form-content input[type="tel"]),
	:global(.dark .form-content input[type="url"]),
	:global(.dark .form-content input[type="search"]),
	:global(.dark .form-content input[type="date"]),
	:global(.dark .form-content input[type="time"]),
	:global(.dark .form-content input[type="datetime-local"]),
	:global(.dark .form-content input[type="month"]),
	:global(.dark .form-content input[type="week"]),
	:global(.dark .form-content input[type="color"]),
	:global(.dark .form-content textarea),
	:global(.dark .form-content select) {
		background-color: #1f2937;
		border-color: #374151;
		color: #f3f4f6;
	}

	:global(.dark .form-content input[type="text"]:focus),
	:global(.dark .form-content input[type="email"]:focus),
	:global(.dark .form-content input[type="password"]:focus),
	:global(.dark .form-content input[type="number"]:focus),
	:global(.dark .form-content input[type="tel"]:focus),
	:global(.dark .form-content input[type="url"]:focus),
	:global(.dark .form-content input[type="search"]:focus),
	:global(.dark .form-content input[type="date"]:focus),
	:global(.dark .form-content input[type="time"]:focus),
	:global(.dark .form-content input[type="datetime-local"]:focus),
	:global(.dark .form-content input[type="month"]:focus),
	:global(.dark .form-content input[type="week"]:focus),
	:global(.dark .form-content input[type="color"]:focus),
	:global(.dark .form-content textarea:focus),
	:global(.dark .form-content select:focus) {
		border-color: #60a5fa;
		box-shadow: 0 0 0 3px rgba(96, 165, 250, 0.1);
	}

	:global(.dark .form-content label) {
		color: #f3f4f6;
	}
</style>
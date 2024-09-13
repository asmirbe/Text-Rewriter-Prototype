<script lang="ts">
	import { onMount, onDestroy } from "svelte";
	import Nav from "$lib/components/Nav.svelte";
	import { debounce } from "lodash-es";
	import TextLoading from "$lib/components/TextLoading.svelte";
	import { Editor } from "@tiptap/core";
	import Placeholder from '@tiptap/extension-placeholder'

	import StarterKit from "@tiptap/starter-kit";
	import { fade } from "svelte/transition";

	let inputText: string = "";
	let outputParagraphs: Array<{ text: string; isComplete: boolean; id: number; originalText: string; isActive: boolean }> = [];
	let isLoading: boolean = false;
	let error: string | null = null;
	let nextParagraphId: number = 0;
	let editor: Editor;
	let activeParagraphIndex: number | null = null;

	let isResizing = false;
	let inputDiv: HTMLElement;
	let outputDiv: HTMLElement;
	let containerDiv: HTMLElement;

	interface ApiResponse {
		response?: string;
	}

	const delay = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

	const rewriteParagraph = async (paragraph: string, id: number): Promise<void> => {
		try {
			await delay(500);

			const response = await fetch("http://localhost:11434/api/generate", {
				method: "POST",
				headers: {
					"Content-Type": "application/json",
				},
				body: JSON.stringify({
					model: "gemma2:2b",
					prompt: `Rewrite the following text in a natural and engaging tone.
                   Maintain the meaning and structure, but use more varied vocabulary,
                   occasional informal language, and adjust phrasing to sound less robotic.
                   Only output the rewritten text, without any additional comments/note.
						 The output text must be at least 90% unique.
						 Do not surround the text with quotation marks or any other special characters.

                   Text to rewrite:
                   ${paragraph}`,
					stream: true,
				}),
			});

			const reader = response.body!.getReader();
			const decoder = new TextDecoder();

			outputParagraphs = outputParagraphs.map((p) => (p.id === id ? { ...p, text: "", isComplete: false } : p));

			while (true) {
				const { done, value } = await reader.read();
				if (done) break;
				const chunk = decoder.decode(value, { stream: true });
				const lines = chunk.split("\n");

				for (const line of lines) {
					if (line.trim() === "") continue;
					const parsed: ApiResponse = JSON.parse(line);
					if (parsed.response && typeof parsed.response === "string") {
						outputParagraphs = outputParagraphs.map((p) => (p.id === id ? { ...p, text: p.text + parsed.response } : p));
					}
				}
			}

			outputParagraphs = outputParagraphs.map((p) => (p.id === id ? { ...p, isComplete: true, originalText: paragraph } : p));
		} catch (error: any) {
			console.error("Error:", error);
			throw new Error("An error occurred while rewriting the paragraph.");
		}
	};

	const processText = async (): Promise<void> => {
		isLoading = true;
		error = null;

		const currentParagraphs = inputText
			.split("\n")
			.map((p) => p.trim())
			.filter((p) => p !== "");

		// Remove paragraphs that no longer exist in the input
		outputParagraphs = outputParagraphs.filter((op, index) => index < currentParagraphs.length);

		for (let i = 0; i < currentParagraphs.length; i++) {
			if (!outputParagraphs[i] || outputParagraphs[i].originalText !== currentParagraphs[i]) {
				const id = outputParagraphs[i]?.id ?? nextParagraphId++;
				const isActive = i === activeParagraphIndex;
				outputParagraphs[i] = { text: "", isComplete: false, id, originalText: currentParagraphs[i], isActive };
				try {
					await rewriteParagraph(currentParagraphs[i], id);
				} catch (err: any) {
					error = err.message;
					outputParagraphs[i] = { text: "Error rewriting paragraph", isComplete: true, id, originalText: currentParagraphs[i], isActive };
				}
			}
		}

		isLoading = false;
	};

	const debouncedProcessText = debounce(processText, 1000);

	const handleInput = () => {
		if (editor) {
			inputText = editor.getText();
			debouncedProcessText();
		}
	};

	const startResize = (event: MouseEvent) => {
		isResizing = true;
		document.addEventListener("mousemove", resize);
		document.addEventListener("mouseup", stopResize);
	};

	const resize = (event: MouseEvent) => {
		if (!isResizing) return;
		const containerRect = containerDiv.getBoundingClientRect();
		const newWidth = ((event.clientX - containerRect.left) / containerRect.width) * 100;
		const minWidth = 30; // Set the minimum width here
		const maxWidth = 70; // Set the maximum width here
		const clampedWidth = Math.max(minWidth, Math.min(maxWidth, newWidth));
		inputDiv.style.width = `${clampedWidth}%`;
		outputDiv.style.width = `${100 - clampedWidth}%`;
	};

	const stopResize = () => {
		isResizing = false;
		document.removeEventListener("mousemove", resize);
		document.removeEventListener("mouseup", stopResize);
	};

	onMount(() => {
		editor = new Editor({
			element: document.querySelector("#editor") as HTMLElement,
			extensions: [
				StarterKit.configure({
					gapcursor: false,
					dropcursor: false,
					bold: false,
					italic: false,
					strike: false,
					code: false,
					codeBlock: false,
					horizontalRule: false,
					blockquote: false,
					heading: false,
					bulletList: false,
					orderedList: false,
					listItem: false,
				}),
				Placeholder.configure({
					placeholder: "Start typing here...",
				}),
			],
			content: "<p></p>",
			autofocus: true,
			editable: true,
			onUpdate: handleInput,
		});

		return () => {
			editor.destroy();
			debouncedProcessText.cancel();
		};
	});

	$: activeParagraphIndex = outputParagraphs.findIndex((p) => p.isActive);


	function getOutputColorStyle(isActive: boolean, isComplete: boolean): string {
		// if (!isComplete) return "color: #B26ADC;";
		return isActive ? "color: #5500B5;" : "color: #B26ADC;";
	}
</script>

<div class="max-w-7xl w-full h-full flex flex-col py-8 px-4 gap-4 mx-auto">
	<!-- <Nav /> -->
	<div bind:this={containerDiv} class="w-full flex min-h-[24rem] gap-4 flex-grow">
		<div bind:this={inputDiv} class="bg-[#fafbfc] shadow-sm text-lg rounded-3xl px-9 py-7 w-1/2 md:w-full flex flex-col gap-6 items-start">
			<div class="px-2 py-0.5 text-xs inline-flex text-black border border-black rounded-full">Editor</div>
			<div id="editor" autocorrect="off" autocapitalize="off" spellcheck="false" class="text-gray-400 outline-none w-full h-full overflow-auto whitespace-pre-wrap" style="white-space: pre-wrap; word-wrap: break-word;"></div>
		</div>
		<div bind:this={outputDiv} id="output" class="bg-[#f9effa] relative h-full rounded-3xl px-9 py-7 border-[#c892e7] border shadow-sm w-1/2 md:w-full flex flex-col gap-6 items-start">
			<!-- svelte-ignore a11y-no-static-element-interactions -->
			<button id="drag" class="absolute left-0 bg-[#f9effa] top-1/2 rounded-full -translate-y-1/2 -translate-x-1/2 h-10 w-2 border border-[#c892e7] cursor-move" on:mousedown|preventDefault|stopPropagation={startResize}></button>
			<div class="px-2 py-0.5 text-xs inline-flex text-[#5500B5] border border-[#5500B5] rounded-full">Co-Pilot</div>
			<div class="overflow-auto">
				{#each outputParagraphs as paragraph}
					<p class="mb-4 text-lg" in:fade out:fade={{ duration: 100 }} style={getOutputColorStyle(paragraph.isActive, paragraph.isComplete)}>
						{paragraph.text}
						{#if !paragraph.isComplete}
							<TextLoading />
						{/if}
					</p>
				{/each}
			</div>
		</div>
	</div>
</div>

<style lang="postcss">
	:global(.ProseMirror) {
		@apply outline-none h-full w-full;
	}
	:global(.tiptap p.is-editor-empty:first-child::before) {
		color: #b3b9be;
		content: attr(data-placeholder);
		float: left;
		height: 0;
		pointer-events: none;
	}
	:global(.ProseMirror p) {
		@apply mb-4;
	}
</style>

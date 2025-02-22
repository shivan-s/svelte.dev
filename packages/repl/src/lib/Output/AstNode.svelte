<script lang="ts">
	import { get_repl_context } from '../context';
	import { tick } from 'svelte';
	import type { CompileResult } from 'svelte/compiler';

	type Ast = CompileResult['ast'];

	export let key = '';
	export let value: Ast;
	export let collapsed = true;
	export let path_nodes: Ast[] = [];
	export let autoscroll = true;

	const { toggleable } = get_repl_context();

	let list_item_el: HTMLLIElement;

	$: is_root = path_nodes[0] === value;
	$: is_leaf = path_nodes[path_nodes.length - 1] === value;
	$: is_ast_array = Array.isArray(value);
	$: is_collapsable = value && typeof value === 'object';
	$: is_markable =
		is_collapsable &&
		'start' in value &&
		'end' in value &&
		typeof value.start === 'number' &&
		typeof value.end === 'number';
	$: key_text = key ? `${key}:` : '';

	let preview_text = '';
	$: {
		if (!is_collapsable || !collapsed) break $;

		if (is_ast_array) {
			if (!('length' in value)) break $;

			preview_text = `[ ${value.length} element${value.length === 1 ? '' : 's'} ]`;
		} else {
			preview_text = `{ ${Object.keys(value).join(', ')} }`;
		}
	}

	$: collapsed = !path_nodes.includes(value);

	$: if (autoscroll && is_leaf && !$toggleable) {
		// wait for all nodes to render before scroll
		tick().then(() => {
			if (list_item_el) {
				list_item_el.scrollIntoView();
			}
		});
	}

	function handle_mark_text(e: MouseEvent | FocusEvent) {
		if (is_markable) {
			e.stopPropagation();

			if (
				'start' in value &&
				'end' in value &&
				typeof value.start === 'number' &&
				typeof value.end === 'number'
			) {
				// TODO
				// $module_editor?.markText({ from: value.start ?? 0, to: value.end ?? 0 });
			}
		}
	}

	function handle_unmark_text(e: MouseEvent) {
		if (is_markable) {
			e.stopPropagation();
			// TODO
			// $module_editor?.unmarkText();
		}
	}
</script>

<li
	bind:this={list_item_el}
	class:marked={!is_root && is_leaf}
	on:mouseover={handle_mark_text}
	on:focus={handle_mark_text}
	on:mouseleave={handle_unmark_text}
>
	{#if !is_root && is_collapsable}
		<button class="ast-toggle" class:open={!collapsed} on:click={() => (collapsed = !collapsed)}>
			{key_text}
		</button>
	{:else if key_text}
		<span>{key_text}</span>
	{/if}
	{#if is_collapsable}
		{#if collapsed && !is_root}
			<button class="preview" on:click={() => (collapsed = !collapsed)}>
				{preview_text}
			</button>
		{:else}
			<span>{is_ast_array ? '[' : '{'}</span>
			<ul>
				{#each Object.entries(value) as [k, v]}
					<svelte:self key={is_ast_array ? '' : k} value={v} {path_nodes} {autoscroll} />
				{/each}
			</ul>
			<span>{is_ast_array ? ']' : '}'}</span>
		{/if}
	{:else}
		<span class="token {typeof value}">
			{JSON.stringify(value)}
		</span>
	{/if}
</li>

<style>
	* {
		font: var(--sk-font-mono);
	}

	ul {
		padding: 0 0 0 2ch;
		margin: 0;
		list-style-type: none;
	}

	.marked {
		background-color: var(--sk-highlight-color);
	}

	button {
		&:hover {
			text-decoration: underline;
		}
	}

	.ast-toggle {
		position: relative;
	}

	.ast-toggle::before {
		content: '\25B6';
		position: absolute;
		bottom: 0;
		left: -1.3rem;
		opacity: 0.7;
	}

	.ast-toggle.open::before {
		content: '\25BC';
	}

	.token {
		color: var(--sk-code-base);
	}

	.token.string {
		color: var(--sk-code-string);
	}

	.token.number {
		color: var(--sk-code-number);
	}
</style>

<script lang="ts">
    import {
        Button,
        InputNumber,
        InputSelect,
        InputText,
        InputTags,
        FormList,
        InputSelectCheckbox
    } from '$lib/elements/forms';
    import { Query } from '@appwrite.io/console';
    import { createEventDispatcher, onMount } from 'svelte';
    import { tags, type Operator, queries, type TagValue } from './store';
    import type { Column } from '$lib/helpers/types';
    import type { Writable } from 'svelte/store';
    import { tooltip } from '$lib/actions/tooltip';

    export let columns: Writable<Column[]>;
    export let columnId: string | null = null;

    const dispatch = createEventDispatcher<{
        clear: void;
        apply: { applied: number };
    }>();

    $: column = $columns.find((c) => c.id === columnId) as Column;
    let arrayValues: string[] = [];

    dispatch('apply', { applied: $tags.length });

    const operators: Record<string, Operator> = {
        'starts with': {
            toQuery: Query.startsWith,
            toTag: (attribute, input) => `**${attribute}** starts with **${input}**`,
            types: ['string']
        },
        'ends with': {
            toQuery: Query.endsWith,
            toTag: (attribute, input) => `**${attribute}** ends with **${input}**`,
            types: ['string']
        },
        'greater than': {
            toQuery: (attr, input) => Query.greaterThan(attr, Number(input)),
            toTag: (attribute, input) => `**${attribute}** greater than **${input}**`,
            types: ['integer', 'double', 'datetime']
        },
        'greater than or equal': {
            toQuery: (attr, input) => Query.greaterThanEqual(attr, Number(input)),
            toTag: (attribute, input) => `**${attribute}** greater than or equal to **${input}**`,
            types: ['integer', 'double', 'datetime']
        },
        'less than': {
            toQuery: Query.lessThan,
            toTag: (attribute, input) => `**${attribute}** less than **${input}**`,
            types: ['integer', 'double', 'datetime']
        },
        'less than or equal': {
            toQuery: Query.lessThanEqual,
            toTag: (attribute, input) => `**${attribute}** less than or equal to **${input}**`,
            types: ['integer', 'double', 'datetime']
        },
        equal: {
            toQuery: Query.equal,
            toTag: (attribute, input) => `**${attribute}** equal to **${input}**`,
            types: ['string', 'integer', 'double', 'boolean']
        },
        'not equal': {
            toQuery: Query.notEqual,
            toTag: (attribute, input) => `**${attribute}** not equal to **${input}**`,
            types: ['string', 'integer', 'double', 'boolean']
        },
        'is not null': {
            toQuery: Query.isNotNull,
            toTag: (attribute) => `**${attribute}** is not null`,
            types: ['string', 'integer', 'double', 'boolean', 'datetime', 'relationship'],
            hideInput: true
        },
        'is null': {
            toQuery: Query.isNull,
            toTag: (attribute) => `**${attribute}** is null`,
            types: ['string', 'integer', 'double', 'boolean', 'datetime', 'relationship'],
            hideInput: true
        },
        contains: {
            toQuery: Query.contains,
            toTag: (attribute, input) => {
                if (Array.isArray(input) && input.length > 2) {
                    return {
                        value: input,
                        tag: `**${attribute}** contains **${formatArray(input)}** `
                    };
                } else {
                    return `**${attribute}** contains **${input}**`;
                }
            },
            types: ['string', 'integer', 'double', 'boolean', 'datetime', 'enum']
        }
    };

    $: operatorsForColumn = Object.entries(operators)
        .filter(([, v]) => v.types.includes(column?.type))
        .map(([k]) => ({
            label: k,
            value: k
        }));

    let operatorKey: string | null = null;
    $: operator = operatorKey ? operators[operatorKey] : null;
    $: {
        columnId;
        operatorKey = null;
    }

    // We cast to any to not cause type errors in the input components
    /* eslint  @typescript-eslint/no-explicit-any: 'off' */
    let value: any = null;

    onMount(() => {
        value = column?.array ? [] : null;
    });

    // This Map is keyed by tags, and has a query as the value
    function addFilter() {
        if (!column || !operator) return;
        if (column.array) {
            queries.addFilter({ column, operator, value: arrayValues });
            columnId = null;
            arrayValues = [];
        } else {
            queries.addFilter({ column, operator, value: value ?? '' });
            columnId = null;
            value = null;
        }
    }

    function tagFormat(node: HTMLElement) {
        node.innerHTML = node.innerHTML.replace(/\*\*(.*?)\*\*/g, '<b>$1</b>');
    }

    function formatArray(array: string[]) {
        if (!array?.length) return;
        if (array.length > 2) {
            return `${array[0]} or ${array.length - 1} others`;
        } else {
            return array.join(' or ');
        }
    }

    function isTypeTagValue(obj: any): obj is TagValue {
        if (typeof obj === 'string') return false;
        return (
            obj &&
            typeof obj.tag === 'string' &&
            (typeof obj.value === 'string' ||
                typeof obj.value === 'number' ||
                Array.isArray(obj.value))
        );
    }

    $: isDisabled = !operator;
</script>

<div>
    <form on:submit|preventDefault={addFilter}>
        <ul class="selects u-flex u-gap-8 u-margin-block-start-16">
            <InputSelect
                id="column"
                options={$columns
                    .filter((c) => c.filter !== false)
                    .map((c) => ({
                        label: c.title,
                        value: c.id
                    }))}
                placeholder="Select column"
                bind:value={columnId} />
            <InputSelect
                id="operator"
                disabled={!column}
                options={operatorsForColumn}
                placeholder="Select operator"
                bind:value={operatorKey} />
        </ul>
        {#if column && operator && !operator?.hideInput}
            {#if column?.array}
                <FormList class="u-margin-block-start-8">
                    {#if column.format === 'enum'}
                        <InputSelectCheckbox
                            name="value"
                            bind:tags={arrayValues}
                            placeholder="Select value"
                            options={column?.elements?.map((value) => ({
                                label: value,
                                value,
                                checked: arrayValues.includes(value)
                            }))}>
                        </InputSelectCheckbox>
                    {:else}
                        <InputTags
                            label="values"
                            showLabel={false}
                            id="value"
                            bind:tags={arrayValues}
                            placeholder="Enter values" />
                    {/if}
                </FormList>
            {:else}
                <ul class="u-margin-block-start-8">
                    {#if column.format === 'enum'}
                        <InputSelect
                            id="value"
                            bind:value
                            placeholder="Select value"
                            options={column?.elements?.map((value) => ({ label: value, value }))}
                            label="Value"
                            showLabel={false} />
                    {:else if column.type === 'integer' || column.type === 'double'}
                        <InputNumber id="value" bind:value placeholder="Enter value" />
                    {:else if column.type === 'boolean'}
                        <InputSelect
                            id="value"
                            placeholder="Select a value"
                            required={true}
                            options={[
                                { label: 'True', value: true },
                                { label: 'False', value: false }
                            ].filter(Boolean)}
                            bind:value />
                    {:else}
                        <InputText id="value" bind:value placeholder="Enter value" />
                    {/if}
                </ul>
            {/if}
        {/if}
        <Button text disabled={isDisabled} class="u-margin-block-start-4" submit>
            <i class="icon-plus" />
            Add filter
        </Button>
    </form>

    <ul class="u-flex u-flex-wrap u-cross-center u-gap-8 u-margin-block-start-16 tags">
        {#each $tags as tag (tag)}
            {#if isTypeTagValue(tag)}
                <button
                    use:tooltip={{
                        content: tag?.value?.toString()
                    }}
                    class="tag"
                    on:click={() => {
                        queries.removeFilter(tag);
                    }}>
                    <span class="text" use:tagFormat>
                        {tag.tag}
                    </span>
                    <i class="icon-x" />
                </button>
            {:else}
                <button
                    class="tag"
                    on:click={() => {
                        queries.removeFilter(tag);
                    }}>
                    <span class="text" use:tagFormat>
                        {tag}
                    </span>
                    <i class="icon-x" />
                </button>
            {/if}
        {/each}
    </ul>
</div>

<style lang="scss">
    .selects {
        :global(> *) {
            flex: 1;
        }
    }

    .tags {
        :global(b) {
            font-weight: bold;
        }
    }
</style>

<svelte:options immutable />

<script context="module" lang="ts">
    export type SearchResultsCapture = number
    interface ResultStateCache {
        count: number
        expanded: Set<SearchMatch>
    }
    const cache = new Map<string, ResultStateCache>()

    const DEFAULT_INITIAL_ITEMS_TO_SHOW = 15
    const INCREMENTAL_ITEMS_TO_SHOW = 10
</script>

<script lang="ts">
    import { mdiCloseOctagonOutline } from '@mdi/js'
    import type { Observable } from 'rxjs'
    import { tick } from 'svelte'

    import { beforeNavigate } from '$app/navigation'
    import Icon from '$lib/Icon.svelte'
    import { observeIntersection } from '$lib/intersection-observer'
    import LoadingSpinner from '$lib/LoadingSpinner.svelte'
    import DynamicFiltersSidebar from '$lib/search/dynamicFilters/Sidebar.svelte'
    import SearchInput from '$lib/search/input/SearchInput.svelte'
    import { submitSearch, type QueryStateStore } from '$lib/search/state'
    import Separator, { getSeparatorPosition } from '$lib/Separator.svelte'
    import { type AggregateStreamingSearchResults, type SearchMatch, type Progress } from '$lib/shared'

    import { getSearchResultComponent } from './searchResultFactory'
    import { setSearchResultsContext } from './searchResultsContext'
    import StreamingProgress from './StreamingProgress.svelte'

    export let stream: Observable<AggregateStreamingSearchResults | undefined>
    export let queryFromURL: string
    export let queryFilters: string
    export let queryState: QueryStateStore

    export function capture(): SearchResultsCapture {
        return resultContainer?.scrollTop ?? 0
    }

    export function restore(capture?: SearchResultsCapture): void {
        if (resultContainer) {
            resultContainer.scrollTop = capture ?? 0
        }
    }

    let resultContainer: HTMLElement | null = null

    const sidebarSize = getSeparatorPosition('search-results-sidebar', 0.2)

    $: progress = $stream?.progress
    // NOTE: done is present but apparently not officially exposed. However
    // $stream.state is always "loading". Need to look into this.
    $: loading = !(progress as Progress & { done?: boolean })?.done
    $: results = $stream?.results

    // Logic for maintaining list state (scroll position, rendered items, open
    // items) for backwards navigation.
    $: cacheEntry = cache.get(queryFromURL)
    $: count = cacheEntry?.count ?? DEFAULT_INITIAL_ITEMS_TO_SHOW
    $: resultsToShow = results ? results.slice(0, count) : null
    $: expandedSet = cacheEntry?.expanded || new Set<SearchMatch>()

    setSearchResultsContext({
        isExpanded(match: SearchMatch): boolean {
            return expandedSet.has(match)
        },
        setExpanded(match: SearchMatch, expanded: boolean): void {
            if (expanded) {
                expandedSet.add(match)
            } else {
                expandedSet.delete(match)
            }
        },
        queryState,
    })
    beforeNavigate(() => {
        cache.set(queryFromURL, { count, expanded: expandedSet })
    })

    function loadMore(event: { detail: boolean }) {
        if (event.detail) {
            count += INCREMENTAL_ITEMS_TO_SHOW
        }
    }

    // FIXME: Not a great solution since it relies on implementation details of
    // the progress component
    async function onResubmitQuery(event: SubmitEvent) {
        const target = event.currentTarget as HTMLElement | null
        const filters = Array.from(target?.querySelectorAll('[name="query"]') ?? [])
            .filter(input => (input as HTMLInputElement).checked)
            .map(input => (input as HTMLInputElement).value)
            .join(' ')
        queryState.setQuery(query => query + ' ' + filters)
        await tick()
        submitSearch($queryState)
    }
</script>

<svelte:head>
    <title>{queryFromURL} - Sourcegraph</title>
</svelte:head>

<div class="search">
    <SearchInput {queryState} />
</div>

<div class="search-results">
    <DynamicFiltersSidebar
        size={$sidebarSize}
        {queryFromURL}
        {queryFilters}
        {queryState}
        streamFilters={$stream?.filters ?? []}
    />
    <Separator currentPosition={sidebarSize} />
    <div class="results" bind:this={resultContainer}>
        <aside class="actions">
            {#if loading}
                <div>
                    <LoadingSpinner inline />
                </div>
            {/if}
            {#if progress}
                <StreamingProgress {progress} on:submit={onResubmitQuery} />
            {/if}
        </aside>
        {#if resultsToShow}
            <ol>
                {#each resultsToShow as result, i}
                    {@const component = getSearchResultComponent(result)}
                    {#if i === resultsToShow.length - 1}
                        <li use:observeIntersection on:intersecting={loadMore}>
                            <svelte:component this={component} {result} />
                        </li>
                    {:else}
                        <li><svelte:component this={component} {result} /></li>
                    {/if}
                {/each}
            </ol>
            {#if resultsToShow.length === 0 && !loading}
                <div class="no-result">
                    <Icon svgPath={mdiCloseOctagonOutline} />
                    <p>No results found</p>
                </div>
            {/if}
        {/if}
    </div>
</div>

<style lang="scss">
    .search {
        border-bottom: 1px solid var(--border-color);
        align-self: stretch;
        padding: 0.25rem;
    }

    .search-results {
        display: flex;
        flex: 1;
        overflow: hidden;
    }

    .results {
        flex: 1;
        overflow: auto;
        display: flex;
        flex-direction: column;

        .actions {
            border-bottom: 1px solid var(--border-color);
            padding: 0.5rem 0;
            padding-left: 0.25rem;
            display: flex;
            align-items: center;
            // Explictly set height to avoid jumping when loading spinner is
            // shown/hidden.
            height: 3rem;
            flex-shrink: 0;
        }

        ol {
            padding: 0;
            margin: 0;
            list-style: none;
        }

        .no-result {
            display: flex;
            flex-direction: column;
            align-items: center;
            margin: auto;
            color: var(--text-muted);
        }
    }
</style>

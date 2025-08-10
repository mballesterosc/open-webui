<script lang="ts">
    import { getContext } from 'svelte';
    import FolderIcon from '../icons/Folder.svelte';
    import ChevronLeft from '../icons/ChevronLeft.svelte';
    import Search from '../icons/Search.svelte';
    import XMark from '../icons/XMark.svelte';
    import Spinner from '../common/Spinner.svelte';

    const i18n = getContext('i18n');

    export let groups: Array<{ id: string | null; name: string }> = [];
    export let items: Array<any> = [];
    export let getGroupName: (groupId: string | null) => string;
    export let filterFn: (item: any, query: string) => boolean;
    export let itemComponent: any; // Svelte component for rendering individual items
    export let query: string = '';
    export let loaded: boolean = false;

    let view: 'folders' | 'items' = 'folders';
    let selectedGroup: { id: string | null; name: string } | null = null;

    $: groupedItems = {};
    $: {
        if (items && groups) {
            const tempGroupedItems = {};
            groups.forEach(group => {
                tempGroupedItems[group.id] = { ...group, items: [] };
            });

            items.forEach(item => {
                const groupId = item.group_id || null; // Use null for public items
                if (tempGroupedItems[groupId]) {
                    tempGroupedItems[groupId].items.push(item);
                } else {
                    // Handle cases where an item might have a group_id not in the provided groups list
                    // This could happen if a group was deleted or not fetched.
                    // For now, we'll put them in a 'Public' or 'Unknown Group' category.
                    if (!tempGroupedItems[null]) {
                        tempGroupedItems[null] = { id: null, name: $i18n.t('Public'), items: [] };
                    }
                    tempGroupedItems[null].items.push(item);
                }
            });

            // Ensure public group exists if there are public items
            if (items.some(item => item.group_id === null) && !tempGroupedItems[null]) {
                tempGroupedItems[null] = { id: null, name: $i18n.t('Public'), items: [] };
                items.filter(item => item.group_id === null).forEach(item => tempGroupedItems[null].items.push(item));
            }
            
            // Filter out empty groups if they were not explicitly defined
            Object.keys(tempGroupedItems).forEach(key => {
                if (tempGroupedItems[key].items.length === 0 && !groups.some(g => g.id === key)) {
                    delete tempGroupedItems[key];
                }
            });

            groupedItems = tempGroupedItems;
        }
    }

    $: filteredGroups = Object.values(groupedItems).filter(group => {
        if (query === '') return true;
        return (group.name || '').toLowerCase().includes(query.toLowerCase());
    });

    $: filteredItemsInSelectedGroup = selectedGroup ? selectedGroup.items.filter(item => filterFn(item, query)) : [];

    function selectFolder(group: { id: string | null; name: string }) {
        selectedGroup = group;
        view = 'items';
    }

    function goBackToFolders() {
        selectedGroup = null;
        view = 'folders';
        query = ''; // Clear search when going back to folders
    }

    function getFolderDisplayName(group: { id: string | null; name: string }) {
        return group.id === null ? $i18n.t('Public') : getGroupName(group.id);
    }
</script>

{#if loaded}
    <div class="flex flex-col gap-1 my-1.5">
        <div class="flex justify-between items-center">
            <div class="flex md:self-center text-xl font-medium px-0.5 items-center">
                {#if view === 'folders'}
                    {$i18n.t('Groups')}
                    <div class="flex self-center w-[1px] h-6 mx-2.5 bg-gray-50 dark:bg-gray-850" />
                    <span class="text-lg font-medium text-gray-500 dark:text-gray-300">
                        {filteredGroups.length}
                    </span>
                {:else if selectedGroup}
                    <button on:click={goBackToFolders} class="p-1.5 rounded-full hover:bg-gray-100 dark:hover:bg-gray-900 transition">
                        <ChevronLeft className="size-4" />
                    </button>
                    <span class="ml-2">
                        {getFolderDisplayName(selectedGroup)}
                    </span>
                    <div class="flex self-center w-[1px] h-6 mx-2.5 bg-gray-50 dark:bg-gray-850" />
                    <span class="text-lg font-medium text-gray-500 dark:text-gray-300">
                        {filteredItemsInSelectedGroup.length}
                    </span>
                {/if}
            </div>
        </div>

        <div class="flex w-full space-x-2">
            <div class="flex flex-1">
                <div class="self-center ml-1 mr-3">
                    <Search className="size-3.5" />
                </div>
                <input
                    class="w-full text-sm py-1 rounded-r-xl outline-hidden bg-transparent"
                    bind:value={query}
                    placeholder={view === 'folders' ? $i18n.t('Search Groups') : $i18n.t('Search Items')}
                />
                {#if query}
                    <div class="self-center pl-1.5 translate-y-[0.5px] rounded-l-xl bg-transparent">
                        <button
                            class="p-0.5 rounded-full hover:bg-gray-100 dark:hover:bg-gray-900 transition"
                            on:click={() => { query = ''; }}
                        >
                            <XMark className="size-3" strokeWidth="2" />
                        </button>
                    </div>
                {/if}
            </div>
        </div>
    </div>

    {#if view === 'folders'}
        <div class="my-2 mb-5 gap-2 grid lg:grid-cols-2 xl:grid-cols-3">
            {#each filteredGroups as group (group.id)}
                <button
                    class="flex flex-col cursor-pointer w-full px-3 py-2 dark:hover:bg-white/5 hover:bg-black/5 rounded-xl transition"
                    on:click={() => selectFolder(group)}
                >
                    <div class="flex items-center gap-4 mt-1 mb-0.5">
                        <div class="w-[44px] h-[44px] flex items-center justify-center">
                            <FolderIcon className="size-8 text-gray-500 dark:text-gray-400" />
                        </div>
                        <div class="flex-1 self-center">
                            <div class="font-semibold line-clamp-1">
                                {getFolderDisplayName(group)}
                            </div>
                            <div class="text-xs text-gray-500 dark:text-gray-400">
                                {$i18n.t('{{count}} items', { count: group.items.length })}
                            </div>
                        </div>
                    </div>
                </button>
            {/each}
            {#if filteredGroups.length === 0 && query !== ''}
                <div class="col-span-full text-center text-gray-500 dark:text-gray-400 mt-4">
                    {$i18n.t('No groups found matching your search.')}
                </div>
            {/if}
        </div>
    {:else if view === 'items' && selectedGroup}
        <div class="my-2 mb-5 gap-2 grid lg:grid-cols-2 xl:grid-cols-3">
            {#each filteredItemsInSelectedGroup as item (item.id)}
                <svelte:component this={itemComponent} {item} />
            {/each}
            {#if filteredItemsInSelectedGroup.length === 0 && query !== ''}
                <div class="col-span-full text-center text-gray-500 dark:text-gray-400 mt-4">
                    {$i18n.t('No items found matching your search in this group.')}
                </div>
            {/if}
        </div>
    {/if}
{:else}
    <div class="w-full h-full flex justify-center items-center">
        <Spinner className="size-5" />
    </div>
{/if}
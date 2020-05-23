<script>
    import {root} from "../state/trackTree";
    import {fade} from "svelte/transition";
    import {configStore} from "../state/settings";
    import {request} from "../broker"
    import colorLookup from "../colors";
    import {get_store_value} from "svelte/internal";

    export let parentStore;
    export let branchStore;
    export let depth;
    export let offset;
    export let parentOffset;

    $: branchState = $branchStore;
    $: path = branchState.path;
    $: childIndex = path[path.length - 1];
    $: pendingLoad = branchState.pendingLoad;

    $: childStores = branchState.children;
    $: children = Object.entries(childStores);

    function leftClick(mouseEvent) {
        if (mouseEvent.button === 0) {
            if (mouseEvent.ctrlKey) {
                if (branchState !== null) {
                    request($configStore, branchStore, branchState);
                }
            } else {
                root.select(path);
            }
        }
    }

    function rightClick() {
        parentStore.deleteChild(childIndex);
    }

    $: onSelectedPath = branchState.onSelectedPath;
    $: selectedByParent = branchState.selectedByParent;
    $: nodeColor = onSelectedPath ? colorLookup.nodeActive : selectedByParent ? colorLookup.nodeWarm : colorLookup.nodeInactive;
    $: edgeColor = onSelectedPath ? colorLookup.edgeActive: selectedByParent ? colorLookup.edgeWarm: colorLookup.edgeInactive;

    $: numberOfLeavesStore = branchStore.numberOfLeavesStore;
    $: numberOfLeaves = $numberOfLeavesStore;

    function getChildPlacements(children, numberOfLeaves) {
        const childPlacements = [];
        let currOffset = offset + (-numberOfLeaves / 2);

        children.forEach(([idx, child]) => {
            const width = get_store_value(child.numberOfLeavesStore);
            childPlacements.push([idx, child, currOffset + (width/2)]);
            currOffset += width;
        });

        return childPlacements;
    }
    $: childPlacements = getChildPlacements(children, numberOfLeaves);

    $: offsetWidth = Math.abs(parentOffset - offset) + 1;
    $: lineWidth = offsetWidth * 60;
    $: lineLeft = Math.min(offset, parentOffset) * 60 - 30;
</script>

<style>
    .node {
        display: flex;
        justify-content: center;
        align-items: center;

        width: 50px;
        height: 50px;
        border-radius: 50%;

        cursor: pointer;
    }

    .label {
        font-size: 30px;
    }

    .pendingLoad {
        font-size: 18px;
        margin: 0;
        text-align: center;
    }

    .line {
        position: absolute;
    }

    .placement {
        position: absolute;
        z-index: 1;
    }
</style>


<div class="placement" style={"top: " + (150*depth) + "px; left: " + (60*offset -25) + "px"}>
    <div
            on:mousedown={leftClick}
            on:contextmenu|preventDefault={rightClick}
            class="node"
            style={"background-color: " + nodeColor}
    >
        <span class="label">
            {childIndex}
        </span>
    </div>
    {#if pendingLoad > 0}
        <p class="pendingLoad" style={"color: " + colorLookup.text}>
                +{pendingLoad}
        </p>
    {/if}
</div>
{#if childPlacements.length > 0}
    {#each childPlacements as [index, child, childOffset] (index)}
        <svelte:self parentStore={branchStore} branchStore={child} depth={depth + 1} offset={childOffset} parentOffset={offset}/>
    {/each}
{/if}
{#if offset < parentOffset}
    <svg class="line" width={lineWidth} height={150} style={"left: " + lineLeft + "px; top: " + (depth * 150 - 125) + "px;"}>
        <line x1="30" x2={lineWidth - 30} y1="150" y2="0" stroke={edgeColor} stroke-width="5px"/>
    </svg>
{:else}
    <svg class="line" width={lineWidth} height={150} style={"left: " + lineLeft + "px; top: " + (depth * 150 - 125) + "px;"}>
        <line x1="30" x2={lineWidth - 30} y1="0" y2="150" stroke={edgeColor} stroke-width="5px"/>
    </svg>
{/if}
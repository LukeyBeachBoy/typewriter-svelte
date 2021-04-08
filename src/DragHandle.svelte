<script lang="ts">
    import {spring} from "svelte/motion";
    import {onDestroy} from "svelte";

    export let element: HTMLElement;
    export let handle: HTMLElement;
    export let dragging = false;
    let previousXPosition;
    let previousYPosition;

    onDestroy(() => {
        window.removeEventListener('mousemove', handleMousemove);
        window.removeEventListener('mouseup', handleMouseup);
    })

    const coords = spring({x: 0, y: 0}, {
        stiffness: 0.2,
        damping: 0.4
    });

    $: if (element) {
        (element as HTMLElement)?.style.transform = `translate(${$coords.x}px, ${$coords.y}px)`;
        handle?.style.transform = `translate(${($coords.x + (element.offsetLeft - 20))}px, ${$coords.y + (element.offsetTop + 5)}px)`;
    }

    async function handleDrag(mouseDownEvent: MouseEvent) {
        dragging = true;
        previousXPosition = mouseDownEvent.clientX;
        previousYPosition = mouseDownEvent.clientY;

        coords.stiffness = coords.damping = 1;

        window.addEventListener('mousemove', handleMousemove);
        window.addEventListener('mouseup', handleMouseup);
    }

    function getHighlightedDropElement() {
        return document.querySelector('.drop-target');
    }

    function handleMousemove(mouseMoveEvent) {
        mouseMoveEvent.preventDefault();
        mouseMoveEvent.stopPropagation();
        const {clientX, clientY} = mouseMoveEvent;
        const dx = clientX - previousXPosition;
        const dy = clientY - previousYPosition;
        previousXPosition = clientX;
        previousYPosition = clientY;

        const highlightedDropElement = getHighlightedDropElement();
        const underCursor = document.elementsFromPoint(mouseMoveEvent.x, mouseMoveEvent.y)
            .find((nearbyElement: HTMLElement) => ![handle, ...handle.children, element].includes(nearbyElement));

        if (underCursor !== highlightedDropElement) {
            highlightedDropElement?.classList.remove('drop-target');
            if (!underCursor.classList.contains('container')) {
                underCursor.classList.add('drop-target');
            }
        }

        coords.update($coords => ({
            x: $coords.x + dx,
            y: $coords.y + dy
        }));
    }

    function handleMouseup(mouseUpEvent) {
        dragging = false;
        const dropTarget = getHighlightedDropElement();

        previousXPosition = mouseUpEvent.clientX;
        previousYPosition = mouseUpEvent.clientY;

        coords.set({
            x: 0,
            y: 0
        })

        if (dropTarget) {
            window.dispatchEvent(new CustomEvent('block-dropped', {
                detail: {
                    targetKey: dropTarget['key'],
                    sourceKey: element['key']
                }
            }))
        }


        dropTarget?.classList.remove('drop-target');
        window.removeEventListener('mousemove', handleMousemove);
        window.removeEventListener('mouseup', handleMouseup);
    }

</script>

<svelte:options accessors/>
<span bind:this={handle} class="drag-handle" on:mousedown|preventDefault|stopPropagation={handleDrag} on:mouseover
      style="cursor: grab; position:absolute; left: -10px; top: 0; visibility: {element ? 'visible' : 'hidden'}; padding-left: 10px">
        <img alt="" src="assets/drag-handle.svg">
    </span>
<script lang="ts">
    import {Delta, Editor, EditorChangeEvent, EditorRange, Source, TextChange,} from "typewriter-editor";
    import {onMount, SvelteComponentTyped} from "svelte";
    import {fromEvent, Observable} from "rxjs";
    import {filter, skip, take, takeWhile} from "rxjs/operators";
    import {UndoStack} from "typewriter-editor/lib/modules/history";
    import DragHandle from './DragHandle.svelte';

    let container: HTMLDivElement;
    let cachedHistory: UndoStack;
    let editor: Editor;
    let localEditor: Editor;
    const dragHandle: SvelteComponentTyped = new DragHandle({target: document.body, props: {element: null}});

    onMount(() => {
        editor = window['editor'] = new Editor({
            root: container,
            html: container.innerHTML,
        });
    });

    function isRedoShortcut(event) {
        return (event.ctrlKey || event.metaKey) && event.key == 'y';
    }

    function isUndoShortcut(event) {
        return (event.ctrlKey || event.metaKey) && event.key == 'z'
    }

    async function handleShortcuts(event: KeyboardEvent) {
        if (localEditor) return;
        if (isUndoShortcut(event) || isRedoShortcut(event)) {
            event.stopPropagation();
            event.preventDefault()
        }

        if (isUndoShortcut(event) && editor.modules.history.hasUndo()) {
            editor.modules.history.undo();
        }

        if (isRedoShortcut(event) && editor.modules.history.hasRedo()) {
            editor.modules.history.redo();
        }

        editor.select(0);
    }

    function enableEditor() {
        editor.setRoot(container);
        if (cachedHistory) {
            editor.modules.history.setStack(cachedHistory);
        }
        editor.enabled = true;
        container.setAttribute('contentEditable', 'false');
    }

    function disableEditor() {
        saveHistory();
        editor.enabled = false;
        const temp = document.createElement('div');
        editor.setRoot(temp);
    }

    function injectLocalEditor(lineRange: EditorRange, cursorPos: number, element: HTMLElement): { localEditorContainer: HTMLDivElement, localChanges: Observable<EditorChangeEvent> } {
        const localEditorContainer = document.createElement('div');
        element.insertAdjacentElement('beforebegin', localEditorContainer);
        element.replaceWith(localEditorContainer)

        localEditor = new Editor({root: localEditorContainer, html: element.outerHTML});
        localEditor.select(cursorPos)
        const localChanges: Observable<EditorChangeEvent> = fromEvent(localEditor, 'change');

        return {localEditorContainer, localChanges};
    }

    function handleClick() {
        if (localEditor) return;
        onSelectionChange();
    }

    function onSelectionChange() {
        const selection = document.getSelection();
        const element = selection.anchorNode.parentElement;
        const lineId = element['key'];
        const lineRange = editor.doc.getLineRange(lineId);
        const changes = [];

        if (!lineRange) return;

        enableEditor();

        const {
            localEditorContainer,
            localChanges
        } = injectLocalEditor(lineRange, selection.focusOffset, element);

        new Promise(requestAnimationFrame).then(() => {
            disableEditor();
            localEditorContainer.focus()
        })

        const changeSubscription = localChanges.subscribe(changeEvent => {
            const change = changeEvent.change.delta;
            if (change.ops.length == 0) return;
            changes.push(change)
        });

        fromEvent(document, 'click').pipe(
            skip(1),
            filter(click => !click.composedPath().includes(localEditorContainer)),
            take(1)
        ).subscribe(_ => {
            changeSubscription.unsubscribe();
            const history = localEditor.modules.history.getStack();
            localEditor.destroy();
            localEditor = null;
            const composed = new Delta(changes.reduce((acc, curr) => {
                return acc.compose(curr)
            }, new Delta()));
            enableEditor();
            editor.update(new Delta().retain(lineRange?.[0]).concat(composed));
            mergeHistory(history, lineRange);
        });
    }

    function retrieveHistory() {
        return cachedHistory;
    }

    function saveHistory() {
        cachedHistory = editor.modules.history.getStack();
    }

    function mergeHistory(historyStack: UndoStack, [changeStartIndex, _]) {
        console.log(historyStack);
        if (historyStack.undo.length === 0 && historyStack.redo.length === 0) return;

        const currentStack: UndoStack = retrieveHistory();
        const undo = historyStack.undo.map(it => {
            let redo: TextChange = it.redo;
            let undo: TextChange = it.undo;

            Object.keys((undo.delta.ops[0])[0] !== 'retain' || undo.delta.ops[0].retain !== changeStartIndex) && changeStartIndex !== 0 ? undo.delta.ops.splice(0, 0, {retain: changeStartIndex}) : null;
            Object.keys((redo.delta.ops[0])[0] !== 'retain' || redo.delta.ops[0].retain !== changeStartIndex) && changeStartIndex !== 0 ? redo.delta.ops.splice(0, 0, {retain: changeStartIndex}) : null;

            return {
                redo,
                undo
            }
        });

        currentStack.undo.pop();
        currentStack.undo = [...currentStack.undo, ...undo];
        currentStack.redo = [];
        editor.modules.history.setStack(currentStack);
    }

    function showDragHandle(event: MouseEvent) {
        if (dragHandle.dragging === true) return;
        const target = event.target as HTMLElement;

        if (target.tagName === 'DIV') return;

        dragHandle.$set({element: target});

        const handleElement = dragHandle.$$.ctx[0];

        fromEvent(document, 'mousemove')
            .pipe(
                filter(evt => ![target, handleElement, ...handleElement.children].includes(evt.target)),
                takeWhile(() => dragHandle.dragging == false),
                take(1)
            ).subscribe(_ => {
            dragHandle.$set({element: null})
        });
    }

    function reorderBlock(event) {
        const {targetKey, sourceKey} = event.detail;

        const sourceBlock = editor.doc.byId[sourceKey];
        const newLocation = editor.doc.getLineRange(targetKey)?.[0];
        const oldLocation = editor.doc.getLineRange(sourceBlock);

        if (newLocation == null) return;

        const delta = new Delta().retain(oldLocation[0]).delete(sourceBlock.length).compose(new Delta().retain(newLocation - sourceBlock.length).concat(sourceBlock.content).insert('\n', sourceBlock.attributes));
        editor.update(delta);
    }
</script>

<style lang="scss">
  * {
    word-break: break-word;
  }

  :focus {
    outline: none;
  }

  .container {
    overflow: hidden;
    padding: 1.5rem;
  }

  .container :global(:not(div)) {
    position: relative;

    &.drop-target {
      &:after {
        border: 1px solid #b8b8ff;
        content: "";
        display: flex;
        position: absolute;
        top: -8px;
        width: 100%;
      }
    }

    &:hover {
      &:before {
        content: "";
        height: 18px;
        left: -5.3px;
        padding: 0 2rem;
        position: absolute;
        top: 7px;
        width: 10px;
      }
    }
  }

</style>

<svelte:window on:block-dropped={reorderBlock} on:keydown|capture={handleShortcuts}/>
<section class="container">
    <div bind:this={container} class="content" on:beforeInput={null} on:click|preventDefault={handleClick}
         on:mouseover|preventDefault={showDragHandle}>
        <h1>Heading 1</h1>
        <p>
            ________________________________________________________________________________________________________________
            ___________________________________________________________________________________________________________________
            ___________________________________________________________________________________________________________________</p>
        <br>
        <h2>Heading 2</h2>
        <p>
            ****************************************************************************************************************
            *******************************************************************************************************************
            *******************************************************************************************************************</p>
    </div>
</section>

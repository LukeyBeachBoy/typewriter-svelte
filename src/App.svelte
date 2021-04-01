<script lang="ts">
    import {Delta, Editor, EditorChangeEvent, EditorRange} from "typewriter-editor";
    import {onMount} from "svelte";
    import {fromEvent, Observable} from "rxjs";
    import {filter, take} from "rxjs/operators";
    import {UndoStack} from "typewriter-editor/lib/modules/history";

    let container: HTMLDivElement;
    let cachedHistory: UndoStack;
    let editor: Editor;
    const defocus = fromEvent(document, 'click');
    window['Delta'] = Delta;


    function isRedoShortcut(event) {
        return (event.ctrlKey || event.metaKey) && event.key == 'y';
    }

    function isUndoShortcut(event) {
        return (event.ctrlKey || event.metaKey) && event.key == 'z'
    }

    async function handleShortcuts(event: KeyboardEvent) {
        console.log('here')
        event.stopPropagation()
        if (!editor.enabled) return;

        if (isUndoShortcut(event) && editor.modules.history.hasUndo()) {
            console.log('undo')
            editor.modules.history.undo();
        }

        if (isRedoShortcut(event) && editor.modules.history.hasRedo()) {
            console.log('redo')
            editor.modules.history.redo();
        }

        await new Promise(requestAnimationFrame);
        editor.select(null);
    }

    function enableEditor() {
        container.setAttribute('contentEditable', 'true');
        editor.setRoot(container);
        editor.enabled = true;
    }

    function disableEditor() {
        container.setAttribute('contentEditable', 'false');
        const hiddenContainer = document.createElement('div');
        editor.setRoot(hiddenContainer);
        editor.enabled = false;
    }

    function injectLocalEditor(lineRange: EditorRange, cursorPos: number, selection: Selection): { localContainer: HTMLDivElement, localEditor: Editor, localChanges: Observable<EditorChangeEvent> } {
        const element = selection.anchorNode.parentElement;

        const localContainer = document.createElement('div');
        element.insertAdjacentElement('beforebegin', localContainer);
        element.replaceWith(localContainer)

        const localEditor: Editor = new Editor({root: localContainer, html: element.outerHTML});
        localEditor.select(cursorPos - lineRange[0]);
        const localChanges = fromEvent(localEditor, 'change');

        return {localContainer, localEditor, localChanges};
    }

    function onSelectionChange(event: EditorChangeEvent) {
        if (event.change?.delta.ops.length === 0) {
            // Selection change
            event.preventDefault();
            event.stopPropagation();

            const selection = document.getSelection();
            const cursorPos = event.change.selection?.[0];
            const lineRange = event.doc.getLineRange(cursorPos);
            const id = editor.doc.getLineAt(cursorPos)?.attributes.id;

            if (!cursorPos) return;

            saveHistory();
            disableEditor();

            const {localContainer, localEditor, localChanges} = injectLocalEditor(lineRange, cursorPos, selection);
            const changeSubscription = localChanges.subscribe(changeEvent => {
                let content: Delta = editor.doc.byId[id].content;
                editor.doc.byId[id].content = content.compose(changeEvent.change.delta);
            });

            defocus.pipe(
                filter(click => !click.composedPath().includes(localContainer)),
                take(1)
            ).subscribe(_ => {
                changeSubscription.unsubscribe();
                localEditor.destroy();
                enableEditor();
                editor.off('change', onSelectionChange);
                mergeHistory(localEditor.modules.history.getStack());
                editor.on('change', onSelectionChange);
            });
        }

    }

    function retrieveHistory() {
        return cachedHistory;
    }

    function saveHistory() {
        cachedHistory = editor.modules.history.getStack();
    }

    function mergeHistory(historyStack: UndoStack) {
        const currentStack: UndoStack = retrieveHistory();
        const undo = historyStack.undo.map(it => {
            let redo = it.redo;
            let undo = it.undo;
            redo.delta.ops = redo.delta.ops.map(op => {
                op.retain ? op.retain += 10 : null;
                return op
            });
            undo.delta.ops = undo.delta.ops.map(op => {
                op.retain ? op.retain += 10 : null;
                return op
            });

            return {
                redo,
                undo
            }
        });

        const redo = historyStack.redo.map(it => {
            let redo = it.redo;
            let undo = it.undo;
            redo.delta.ops = redo.delta.ops.map(op => {
                op.retain ? op.retain += 10 : null;
                return op
            });
            undo.delta.ops = undo.delta.ops.map(op => {
                op.retain ? op.retain += 10 : null;
                return op
            });

            return {
                redo,
                undo
            }
        })

        currentStack.undo = [...currentStack.undo, ...undo];
        currentStack.redo = [...currentStack.redo, ...redo];
        editor.modules.history.setStack(currentStack);

    }

    onMount(() => {
        editor = window['editor'] = new Editor({
            root: container,
            html: container.innerHTML,
        });
        editor.on('change', onSelectionChange);
    });
</script>

<style lang="scss">
  :focus {
    outline: none;
  }
</style>

<svelte:body on:keydown={handleShortcuts}/>
<div bind:this={container} on:beforeInput={null}>
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

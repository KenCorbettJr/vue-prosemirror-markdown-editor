<template>
  <div class="recipe-steps-editor">
		<div ref="wrapper"></div>
	</div>
</template>

<script>
import Vue from 'vue';

import { EditorState, Plugin, PluginKey } from "prosemirror-state";
import { EditorView, Decoration, DecorationSet } from "prosemirror-view";
import { keymap } from "prosemirror-keymap";
import { history, undo, redo } from "prosemirror-history";
import { baseKeymap } from "prosemirror-commands";
import { schema, defaultMarkdownParser, defaultMarkdownSerializer } from "prosemirror-markdown";

const placeholderPlugin = new Plugin({
  key: new PluginKey("placeholder"),
  state: {
    init: () => DecorationSet.empty,
    apply(tr, set) {
      // Adjust decoration positions to changes made by the transaction
      set = set.map(tr.mapping, tr.doc)
      // See if the transaction adds or removes any placeholders
      let action = tr.getMeta(this)
      if (action && action.add) {
        let image = document.createElement("img")
        image.src = action.add.src;
        image.className = "placeholder-image";
        let deco = Decoration.widget(action.add.pos, image, {id: action.add.id})
        set = set.add(tr.doc, [deco])
      } else if (action && action.remove) {
        set = set.remove(set.find(null, null,
                                  spec => spec.id == action.remove.id))
      }
      return set
    }
  },
  props: {
    decorations(state) {
      return this.getState(state)
    }
  }
})

export default {
    name: "MarkdownEditor",
    props: ['value'],
    data() {
        return {
            content: this.value
        };
    },
    methods: {
        placeholderFocused: function placeholderFocused() {
            this.initializeProsemirror();
        },
        initializeProsemirror: function initializeProsemirror() {
            this.showPlaceholder = false;

            Vue.nextTick(() => {
                const wrapper = this.$refs.wrapper;
                const doc = defaultMarkdownParser.parse(this.content);

                this.view = new EditorView(wrapper, {
                    state: EditorState.create({
                        doc,
                        plugins: [
                            placeholderPlugin,
                            history(),
                            keymap(baseKeymap),
                            keymap({
                                "Mod-z": undo,
                                "Mod-y": redo
                            })
                        ],
                    }),
                    handlePaste: (view, event, slice) => {
                        if (event.clipboardData && event.clipboardData.items && event.clipboardData.items.length > 0) {
                            [...event.clipboardData.items].forEach((item) => {
                                if (item.type.indexOf("image") !== -1) {
                                    const id = Math.floor(Math.random() * 1000000);
                                    const file = item.getAsFile();
                                    this.uploadImage(file).then(url => {
                                        let pos = this.findPlaceholder(id);
                                        if (pos == null) return;
                                        const image = schema.node("image", {src:url})
                                        const addImageTransaction = this.view.state.tr
                                          .replaceWith(pos, pos, image)
                                          .setMeta(placeholderPlugin, {remove: {id}});
                                        this.view.dispatch(addImageTransaction);
                                    });
                                    this.convertFileToDataUrl(file).then(url => {
                                        const tr = this.view.state.tr
                                        if (!tr.selection.empty) {
                                          tr.deleteSelection();
                                        }
                                        tr.setMeta(placeholderPlugin, {add: {id, pos: tr.selection.from, src: url}})
                                        this.view.dispatch(tr);
                                    });
                                }
                            });
                        }
                    },
                    dispatchTransaction: (transaction) => {
                        this.view.updateState(this.view.state.apply(transaction));
                        this.content = defaultMarkdownSerializer.serialize(this.view.state.doc);
                        this.$emit('input', this.content);
                    }
                });
                wrapper.children[0].focus();
            });
        },
        convertFileToDataUrl: function(file) {
            return new Promise((resolve) => {
                const reader = new FileReader();
                reader.addEventListener('loadend', () => {
                    resolve(reader.result);
                });
                reader.readAsDataURL(file);
            })
        },
        findPlaceholder: function(id) {
            let decos = this.view.state["placeholder$"];
            let found = decos && decos.find(null, null, spec => spec.id == id);
            return found && found.length ? found[0].from : null
        },
        uploadImage: function(file) {
            // TODO: Upload Image Here

            // The following is just throw away code to convert the file to a url and add a bit of delay
            return new Promise(resolve => {
                this.convertFileToDataUrl(file).then((url) => {
                    setTimeout(() => resolve(url), 2000);
                });
            });
        },
    },
    mounted(){
        this.initializeProsemirror();
    }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
  .ProseMirror {
    min-height: 1em;
    background: #f8f8f8;
    padding: 1em;
    width: 100%;
  }

  .placeholder-image {
    opacity: .5;
  }
</style>

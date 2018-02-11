<template>
	<div id="app">
		<markdown-editor v-model="comment" />
	</div>
</template>

<script>
import markdownEditor from './components/MarkdownEditor';

export default {
	name: 'app',
	components: {
		markdownEditor
	},
	data () {
    return {
			comment: `# H1 Heading
	\n### H2 Heading
	\n[link](www.google.com)`,
    };
	},
}
</script>

<style>
	#app {
		font-family: 'Avenir', Helvetica, Arial, sans-serif;
		-webkit-font-smoothing: antialiased;
		-moz-osx-font-smoothing: grayscale;
		color: #2c3e50;
	}
</style>
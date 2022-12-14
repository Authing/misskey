<template>
<div v-if="page" class="iroscrza" :class="{ shadow: $store.state.device.useShadow, round: $store.state.device.roundedCorners, center: page.alignCenter }" :style="{ fontFamily: page.font }" :key="path">
	<header>
		<div class="title">{{ page.title }}</div>
	</header>

	<div v-if="script">
		<x-block v-for="child in page.content" :value="child" @input="v => updateBlock(v)" :page="page" :script="script" :key="child.id" :h="2"/>
	</div>

	<footer>
		<small>@{{ page.user.username }}</small>
		<router-link v-if="$store.getters.isSignedIn && $store.state.i.id === page.userId" :to="`/i/pages/edit/${page.id}`">{{ $t('edit-this-page') }}</router-link>
		<router-link :to="`./${page.name}/view-source`">{{ $t('view-source') }}</router-link>
		<div class="like">
			<button @click="unlike()" v-if="page.isLiked" :title="$t('unlike')"><fa :icon="faHeartS"/></button>
			<button @click="like()" v-else :title="$t('like')"><fa :icon="faHeart"/></button>
			<span class="count" v-if="page.likedCount > 0">{{ page.likedCount }}</span>
		</div>
	</footer>
</div>
</template>

<script lang="ts">
import Vue from 'vue';
import i18n from '../../../../i18n';
import { faHeart as faHeartS } from '@fortawesome/free-solid-svg-icons';
import { faHeart, faStickyNote } from '@fortawesome/free-regular-svg-icons';
import XBlock from './page.block.vue';
import { ASEvaluator } from '../../../../../../misc/aiscript/evaluator';
import { collectPageVars } from '../../../scripts/collect-page-vars';
import { url } from '../../../../config';

class Script {
	public aiScript: ASEvaluator;
	private onError: any;
	public vars: Record<string, any>;

	constructor(aiScript, onError) {
		this.aiScript = aiScript;
		this.onError = onError;
		this.eval();
	}

	public eval() {
		try {
			this.vars = this.aiScript.evaluateVars();
		} catch (e) {
			this.onError(e);
		}
	}

	public interpolate(str: string) {
		if (str == null) return null;
		return str.replace(/{(.+?)}/g, match => {
			const v = this.vars[match.slice(1, -1).trim()];
			return v == null ? 'NULL' : v.toString();
		});
	}
}

export default Vue.extend({
	i18n: i18n('pages'),

	components: {
		XBlock
	},

	props: {
		pageName: {
			type: String,
			required: true
		},
		username: {
			type: String,
			required: true
		},
	},

	data() {
		return {
			page: null,
			script: null,
			faHeartS, faHeart
		};
	},

	computed: {
		path(): string {
			return this.username + '/' + this.pageName;
		}
	},

	watch: {
		path() {
			this.fetch();
		}
	},

	created() {
		this.fetch();
	},

	methods: {
		fetch() {
			this.$root.api('pages/show', {
				name: this.pageName,
				username: this.username,
			}).then(page => {
				this.page = page;
				this.$emit('init', {
					title: this.page.title,
					icon: faStickyNote
				});
				const pageVars = this.getPageVars();
				this.script = new Script(new ASEvaluator(this.page.variables, pageVars, {
					randomSeed: Math.random(),
					user: page.user,
					visitor: this.$store.state.i,
					page: page,
					url: url
				}), e => {
					console.dir(e);
				});
			});
		},

		getPageVars() {
			return collectPageVars(this.page.content);
		},

		like() {
			this.$root.api('pages/like', {
				pageId: this.page.id,
			}).then(() => {
				this.page.isLiked = true;
				this.page.likedCount++;
			});
		},

		unlike() {
			this.$root.api('pages/unlike', {
				pageId: this.page.id,
			}).then(() => {
				this.page.isLiked = false;
				this.page.likedCount--;
			});
		}
	}
});
</script>

<style lang="stylus" scoped>
.iroscrza
	overflow hidden
	background var(--face)

	&.center
		text-align center

	&.round
		border-radius 6px

	&.shadow
		box-shadow 0 3px 8px rgba(0, 0, 0, 0.2)

	> header
		> .title
			z-index 1
			margin 0
			padding 16px 32px
			font-size 20px
			font-weight bold
			color var(--text)
			box-shadow 0 var(--lineWidth) rgba(#000, 0.07)

			@media (max-width 600px)
				padding 16px 32px
				font-size 20px

			@media (max-width 400px)
				padding 10px 20px
				font-size 16px

	> div
		color var(--text)
		padding 24px 32px
		font-size 16px

		@media (max-width 600px)
			padding 24px 32px
			font-size 16px

		@media (max-width 400px)
			padding 20px 20px
			font-size 15px

	> footer
		color var(--text)
		padding 0 32px 28px 32px

		@media (max-width 600px)
			padding 0 32px 28px 32px

		@media (max-width 400px)
			padding 0 20px 20px 20px
			font-size 14px

		> small
			display block
			opacity 0.5

		> a
			font-size 90%

		> a + a
			margin-left 8px

		> .like
			margin-top 16px

</style>

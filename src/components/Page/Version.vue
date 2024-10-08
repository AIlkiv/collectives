<!--
  - SPDX-FileCopyrightText: 2021 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

<template>
	<div :class="[isFullWidthView ? 'full-width-view' : 'sheet-view']">
		<h2 id="titleform" class="page-title">
			<div class="page-title-icon">
				<div v-if="currentPage.emoji">
					{{ currentPage.emoji }}
				</div>
				<EmoticonOutlineIcon v-else
					class="emoji-picker-emoticon"
					:size="pageTitleIconSize"
					fill-color="var(--color-text-maxcontrast)" />
			</div>

			<input class="title title-version"
				:class="{ 'mobile': isMobile }"
				type="text"
				disabled
				:value="versionTitle">
			<NcButton :title="t('collectives', 'Restore this version')"
				:aria-label="t('collectives', 'Restore this version')"
				class="titleform-button"
				@click="revertVersion">
				<template #icon>
					<RestoreIcon :size="20" />
				</template>
				{{ t('collectives', 'Restore') }}
			</NcButton>
			<NcActions>
				<NcActionButton :close-after-click="true" @click="closeVersions">
					<template #icon>
						<DockRightIcon :size="20" />
					</template>
				</NcActionButton>
			</NcActions>
		</h2>
		<SkeletonLoading v-show="!contentLoaded" class="page-content-skeleton" type="text" />
		<div v-show="contentLoaded"
			id="text-container">
			<div ref="reader" data-collectives-el="reader" />
		</div>
	</div>
</template>

<script>
import { NcActionButton, NcActions, NcButton } from '@nextcloud/vue'
import DockRightIcon from 'vue-material-design-icons/DockRight.vue'
import RestoreIcon from 'vue-material-design-icons/Restore.vue'

import { getCurrentUser } from '@nextcloud/auth'
import axios from '@nextcloud/axios'
import { showError, showSuccess } from '@nextcloud/dialogs'
import { generateRemoteUrl } from '@nextcloud/router'
import { mapActions, mapState } from 'pinia'
import { useRootStore } from '../../stores/root.js'
import { usePagesStore } from '../../stores/pages.js'
import { useVersionsStore } from '../../stores/versions.js'
import isMobile from '@nextcloud/vue/dist/Mixins/isMobile.js'
import EmoticonOutlineIcon from 'vue-material-design-icons/EmoticonOutline.vue'
import editorMixin from '../../mixins/editorMixin.js'
import pageContentMixin from '../../mixins/pageContentMixin.js'
import SkeletonLoading from '../SkeletonLoading.vue'

export default {
	name: 'Version',

	components: {
		DockRightIcon,
		EmoticonOutlineIcon,
		NcActionButton,
		NcActions,
		NcButton,
		RestoreIcon,
		SkeletonLoading,
	},

	mixins: [
		editorMixin,
		isMobile,
		pageContentMixin,
	],

	computed: {
		...mapState(useRootStore, ['loading']),
		...mapState(useVersionsStore, ['version']),
		...mapState(usePagesStore, [
			'currentPage',
			'isFullWidthView',
			'title',
		]),

		pageTitleIconSize() {
			return isMobile ? 25 : 30
		},

		restoreFolderUrl() {
			return generateRemoteUrl(
				`dav/versions/${this.getUser}/restore/${this.currentPage.id}`,
			)
		},

		getUser() {
			return getCurrentUser().uid
		},

		versionTitle() {
			return `${this.title} (${this.version.relativeTimestamp})`
		},

		contentLoaded() {
			return !!this.davContent || !this.loading(`version-${this.currentPage.id}-${this.version.timestamp}`)
		},
	},

	watch: {
		'version.timestamp'() {
			this.davContent = ''
			this.getPageContent()
		},
	},

	mounted() {
		this.pageInfoBarPage = {}
		this.setupReader()
		this.getPageContent()
	},

	methods: {
		...mapActions(useRootStore, ['done', 'hide', 'load']),
		...mapActions(useVersionsStore, ['getVersions', 'selectVersion']),

		closeVersions() {
			this.selectVersion(null)
			this.hide('sidebar')
		},
		/**
		 * Revert page to an old version
		 */
		async revertVersion() {
			const target = this.version
			try {
				await axios({
					method: 'MOVE',
					url: target.downloadUrl,
					headers: {
						Destination: this.restoreFolderUrl,
					},
				})
				this.selectVersion(null)
				this.getVersions(this.currentPage.id)
				showSuccess(t('collectives', 'Reverted {page} to revision {timestamp}.', {
					page: this.currentPage.title,
					timestamp: target.relativeTimestamp,
				}))
			} catch (e) {
				showError(t('collectives', 'Failed to revert {page} to revision {timestamp}.', {
					page: this.currentPage.title,
					timestamp: target.relativeTimestamp,
				}))
				console.error('Failed to move page to restore folder', e)
			}
		},

		async getPageContent() {
			this.load(`version-${this.currentPage.id}-${this.version.timestamp}`)
			this.davContent = await this.fetchPageContent(this.version.downloadUrl)
			this.reader?.setContent(this.davContent)
			this.done(`version-${this.currentPage.id}-${this.version.timestamp}`)
		},
	},
}
</script>

<style lang="scss" scoped>
input[type="text"].title-version {
	color: var(--color-text-maxcontrast);
}

.page-content-skeleton {
	padding-top: var(--default-clickable-area);
}
</style>

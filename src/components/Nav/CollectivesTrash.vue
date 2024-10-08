<!--
  - SPDX-FileCopyrightText: 2022 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

<template>
	<div id="collectives-trash"
		v-click-outside="closeTrash"
		:class="{ open }">
		<div id="collectives-trash__header">
			<NcButton type="tertiary" class="collectives-trash-button" @click="toggleTrash">
				<template #icon>
					<DeleteIcon class="collectives-trash-button__icon" :size="20" />
				</template>
				{{ t('collectives', 'Deleted collectives') }}
			</NcButton>
		</div>
		<transition name="slide-up">
			<div v-show="open" id="collectives-trash__content">
				<ul class="app-navigation__list">
					<NcAppNavigationItem v-for="collective in sortedTrashCollectives"
						:key="collective.circleId"
						:name="collective.name"
						:force-menu="true"
						:force-display-actions="isMobile"
						class="collectives_trash_list_item">
						<template #icon>
							<template v-if="collective.emoji">
								{{ collective.emoji }}
							</template>
							<template v-else>
								<CollectivesIcon :size="20" />
							</template>
						</template>
						<template #actions>
							<NcActionButton :close-after-click="true" @click="restoreCollective(collective)">
								<template #icon>
									<RestoreIcon :size="20" />
								</template>
								{{ t('collectives', 'Restore') }}
							</NcActionButton>
							<NcActionButton :close-after-click="true" @click="showDeleteModal(collective)">
								<template #icon>
									<DeleteIcon :size="20" />
								</template>
								{{ t('collectives', 'Delete permanently') }}
							</NcActionButton>
						</template>
					</NcAppNavigationItem>
				</ul>
			</div>
		</transition>

		<NcDialog v-if="deleteModal"
			:name="t('collectives', 'Permanently delete collective {collective}', { collective: modalCollective.name })"
			size="small"
			@closing="closeDeleteModal">
			<div class="modal__content">
				<div>
					{{ t('collectives', 'Delete corresponding team along with the collective?') }}
				</div>
			</div>
			<template #actions>
				<NcButton type="error"
					:wide="true"
					@click="deleteCollective(modalCollective, false)">
					{{ t('collectives', 'Only collective') }}
				</NcButton>
				<NcButton v-if="isCollectiveOwner(modalCollective)"
					type="error"
					:wide="true"
					@click="deleteCollective(modalCollective, true)">
					{{ t('collectives', 'Collective and team') }}
				</NcButton>
				<NcButton v-else
					type="primary"
					disabled
					:title="t('collectives', 'Only team owners can delete a team')"
					:wide="true">
					{{ t('collectives', 'Collective and team') }}
				</NcButton>
				<NcButton :wide="true" @click="closeDeleteModal">
					{{ t('collectives', 'Cancel') }}
				</NcButton>
			</template>
		</NcDialog>
	</div>
</template>

<script>
import { mapState } from 'pinia'
import { useCollectivesStore } from '../../stores/collectives.js'
import { NcActionButton, NcAppNavigationItem, NcButton, NcDialog } from '@nextcloud/vue'
import isMobile from '@nextcloud/vue/dist/Mixins/isMobile.js'
import CollectivesIcon from '../Icon/CollectivesIcon.vue'
import DeleteIcon from 'vue-material-design-icons/Delete.vue'
import RestoreIcon from 'vue-material-design-icons/Restore.vue'
import { directive as ClickOutside } from 'v-click-outside'

export default {
	name: 'CollectivesTrash',

	directives: {
		ClickOutside,
	},

	components: {
		NcActionButton,
		NcAppNavigationItem,
		NcButton,
		CollectivesIcon,
		DeleteIcon,
		NcDialog,
		RestoreIcon,
	},

	mixins: [
		isMobile,
	],

	data() {
		return {
			open: false,
			deleteModal: false,
			modalCollective: null,
		}
	},

	computed: {
		...mapState(useCollectivesStore, [
			'isCollectiveOwner',
			'sortedTrashCollectives',
		]),
	},

	methods: {
		toggleTrash() {
			this.open = !this.open
		},
		closeTrash() {
			this.open = false
		},
		restoreCollective(collective) {
			this.$emit('restore-collective', collective)
		},
		deleteCollective(collective, circle) {
			this.$emit('delete-collective', collective, circle)
			this.closeDeleteModal()
		},
		showDeleteModal(collective) {
			this.modalCollective = collective
			this.deleteModal = true
		},
		closeDeleteModal() {
			this.modalCollective = null
			this.deleteModal = false
		},
	},
}
</script>

<style lang="scss" scoped>
#collectives-trash {
	margin-top: auto;
	padding: 3px;

	&__header {
		box-sizing: border-box;
		margin: 0 3px 3px 3px;
		padding-top: calc(var(--default-grid-baseline, 4px) * 2);

		.collectives-trash-button {
			display: flex;
			flex: 1 1 0;
			height: var(--default-clickable-area);
			width: 100%;
			padding: 0;
			margin: 0;
			background-color: var(--color-main-background);
			box-shadow: none;
			border: 0;
			border-radius: var(--border-radius-element, var(--border-radius-large));
			// text-align: left;
			// font-weight: normal;
			// font-size: 100%;
			color: var(--color-main-text);
			padding-right: 14px;
			line-height: var(--default-clickable-area);

			:deep(.button-vue__wrapper) {
				width: 100%;
				justify-content: start;
			}

			&:hover,
			&:focus {
				background-color: var(--color-background-hover);
			}

			&__icon {
				width: var(--default-clickable-area);
				height: var(--default-clickable-area);
				min-width: var(--default-clickable-area);
			}

			:deep(.button-vue__text) {
				overflow: hidden;
				white-space: nowrap;
				text-overflow: ellipsis;
				font-weight: normal;
			}
		}
	}

	&__content {
		display: block;
		padding: 10px;
		/* Restrict height of trash an make scrollable */
		max-height: 300px;
		overflow-y: auto;
		box-sizing: border-box;
	}

	.slide-up-leave-active,
	.slide-up-enter-active {
		transition-duration: var(--animation-slow);
		transition-property: max-height, padding;
		overflow-y: hidden !important;
	}

	.slide-up-enter,
	.slide-up-leave-to {
		max-height: 0 !important;
		padding: 0 10px !important;
	}
}

:deep(.modal-wrapper--small) {
	.modal-container {
		width: 600px;
	}
}

.modal__content {
	margin: 15px;
}

:deep(.dialog__actions) {
	display: flex;
	flex-flow: column nowrap;
	gap: 10px;
}
</style>

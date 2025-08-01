<script setup lang="ts">
import Modal from '@/components/Modal.vue';
import { COMMUNITY_PACKAGE_CONFIRM_MODAL_KEY, COMMUNITY_PACKAGE_MANAGE_ACTIONS } from '@/constants';
import { useToast } from '@/composables/useToast';
import { useCommunityNodesStore } from '@/stores/communityNodes.store';
import { createEventBus } from '@n8n/utils/event-bus';
import { useI18n } from '@n8n/i18n';
import { useTelemetry } from '@/composables/useTelemetry';
import { computed, onMounted, ref } from 'vue';
import { useNodeTypesStore } from '@/stores/nodeTypes.store';
import type { CommunityNodeType } from '@n8n/api-types';
import { useSettingsStore } from '@/stores/settings.store';
import semver from 'semver';
import { N8nText } from '@n8n/design-system';
import { useUIStore } from '@/stores/ui.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import type { WorkflowResource } from '@/Interface';

export type CommunityPackageManageMode = 'uninstall' | 'update' | 'view-documentation';

interface Props {
	modalName: string;
	activePackageName: string;
	mode: CommunityPackageManageMode;
}

const props = defineProps<Props>();

const communityNodesStore = useCommunityNodesStore();
const nodeTypesStore = useNodeTypesStore();
const settingsStore = useSettingsStore();
const workflowsStore = useWorkflowsStore();

const modalBus = createEventBus();

const toast = useToast();
const i18n = useI18n();
const telemetry = useTelemetry();

const loading = ref(false);

const workflowsWithPackageNodes = ref<WorkflowResource[]>([]);

const isUsingVerifiedAndUnverifiedPackages =
	settingsStore.isCommunityNodesFeatureEnabled && settingsStore.isUnverifiedPackagesEnabled;
const isUsingVerifiedPackagesOnly =
	settingsStore.isCommunityNodesFeatureEnabled && !settingsStore.isUnverifiedPackagesEnabled;

const communityStorePackage = computed(
	() => communityNodesStore.installedPackages[props.activePackageName],
);
const updateVersion = computed(() => {
	return settingsStore.isUnverifiedPackagesEnabled
		? communityStorePackage.value.updateAvailable
		: nodeTypeStorePackage.value?.npmVersion;
});
const nodeTypeStorePackage = ref<CommunityNodeType>();

const isLatestPackageVerified = ref<boolean>(true);

const packageVersion = ref<string>(communityStorePackage.value.updateAvailable ?? '');

const includedNodes = computed(() => {
	return communityStorePackage.value.installedNodes.map((node) => node.name).join(', ');
});

const getModalContent = computed(() => {
	if (props.mode === COMMUNITY_PACKAGE_MANAGE_ACTIONS.UNINSTALL) {
		return {
			title: i18n.baseText('settings.communityNodes.confirmModal.uninstall.title'),
			message: i18n.baseText('settings.communityNodes.confirmModal.includedNodes', {
				interpolate: {
					nodes: includedNodes.value,
				},
			}),
			description: workflowsWithPackageNodes.value.length
				? i18n.baseText('settings.communityNodes.confirmModal.uninstall.description')
				: i18n.baseText('settings.communityNodes.confirmModal.noWorkflowsUsingNodes'),
			buttonLabel: i18n.baseText('settings.communityNodes.confirmModal.uninstall.buttonLabel'),
			buttonLoadingLabel: i18n.baseText(
				'settings.communityNodes.confirmModal.uninstall.buttonLoadingLabel',
			),
		};
	}
	return {
		title: i18n.baseText('settings.communityNodes.confirmModal.update.title'),
		message: i18n.baseText('settings.communityNodes.confirmModal.includedNodes', {
			interpolate: {
				nodes: includedNodes.value,
			},
		}),
		description: workflowsWithPackageNodes.value.length
			? i18n.baseText('settings.communityNodes.confirmModal.update.description')
			: i18n.baseText('settings.communityNodes.confirmModal.noWorkflowsUsingNodes'),
		warning: i18n.baseText('settings.communityNodes.confirmModal.update.warning'),
		buttonLabel: i18n.baseText('settings.communityNodes.confirmModal.update.buttonLabel'),
		buttonLoadingLabel: i18n.baseText(
			'settings.communityNodes.confirmModal.update.buttonLoadingLabel',
		),
	};
});

const onModalClose = () => {
	return !loading.value;
};

const onConfirmButtonClick = async () => {
	if (props.mode === COMMUNITY_PACKAGE_MANAGE_ACTIONS.UNINSTALL) {
		await onUninstall();
	} else if (props.mode === COMMUNITY_PACKAGE_MANAGE_ACTIONS.UPDATE) {
		await onUpdate();
	}
};

const onUninstall = async () => {
	try {
		telemetry.track('user started cnr package deletion', {
			package_name: communityStorePackage.value.packageName,
			package_node_names: communityStorePackage.value.installedNodes.map((node) => node.name),
			package_version: communityStorePackage.value.installedVersion,
			package_author: communityStorePackage.value.authorName,
			package_author_email: communityStorePackage.value.authorEmail,
		});
		loading.value = true;
		await communityNodesStore.uninstallPackage(props.activePackageName);
		await useNodeTypesStore().getNodeTypes();
		toast.showMessage({
			title: i18n.baseText('settings.communityNodes.messages.uninstall.success.title'),
			type: 'success',
		});
	} catch (error) {
		toast.showError(error, i18n.baseText('settings.communityNodes.messages.uninstall.error'));
	} finally {
		loading.value = false;
		modalBus.emit('close');
	}
};

const onUpdate = async () => {
	try {
		telemetry.track('user started cnr package update', {
			package_name: communityStorePackage.value.packageName,
			package_node_names: communityStorePackage.value.installedNodes.map((node) => node.name),
			package_version_current: communityStorePackage.value.installedVersion,
			package_version_new: communityStorePackage.value.updateAvailable,
			package_author: communityStorePackage.value.authorName,
			package_author_email: communityStorePackage.value.authorEmail,
		});
		loading.value = true;

		if (settingsStore.isUnverifiedPackagesEnabled) {
			await communityNodesStore.updatePackage(props.activePackageName);
		} else if (settingsStore.isCommunityNodesFeatureEnabled) {
			await communityNodesStore.updatePackage(
				props.activePackageName,
				updateVersion.value,
				nodeTypeStorePackage.value?.checksum,
			);
		} else {
			throw new Error('Community nodes feature is not correctly enabled.');
		}

		await useNodeTypesStore().getNodeTypes();
		toast.showMessage({
			title: i18n.baseText('settings.communityNodes.messages.update.success.title'),
			message: i18n.baseText('settings.communityNodes.messages.update.success.message', {
				interpolate: {
					packageName: props.activePackageName,
					version: updateVersion.value ?? '',
				},
			}),
			type: 'success',
		});
	} catch (error) {
		toast.showError(error, i18n.baseText('settings.communityNodes.messages.update.error.title'));
	} finally {
		loading.value = false;
		modalBus.emit('close');
	}
};

async function fetchPackageInfo(packageName: string) {
	await nodeTypesStore.loadNodeTypesIfNotLoaded();
	const nodeType = nodeTypesStore.visibleNodeTypes.find((nodeType) =>
		nodeType.name.includes(packageName),
	);

	if (nodeType) {
		const communityNodeAttributes = await nodeTypesStore.getCommunityNodeAttributes(nodeType?.name);

		nodeTypeStorePackage.value = communityNodeAttributes ?? undefined;
	}
}

function setIsVerifiedLatestPackage() {
	if (
		isUsingVerifiedAndUnverifiedPackages &&
		nodeTypeStorePackage.value?.npmVersion &&
		communityStorePackage.value.updateAvailable
	) {
		isLatestPackageVerified.value = semver.eq(
			nodeTypeStorePackage.value.npmVersion,
			communityStorePackage.value.updateAvailable,
		);
	}
}

function setPackageVersion() {
	if (isUsingVerifiedPackagesOnly) {
		packageVersion.value = nodeTypeStorePackage.value?.npmVersion ?? packageVersion.value;
	}
}

const onClick = async () => {
	useUIStore().closeModal(COMMUNITY_PACKAGE_CONFIRM_MODAL_KEY);
};

onMounted(async () => {
	if (props.activePackageName) {
		await fetchPackageInfo(props.activePackageName);
	}

	if (communityStorePackage.value?.installedNodes.length) {
		const nodeTypes = communityStorePackage.value.installedNodes.map((node) => node.type);
		const response = await workflowsStore.fetchWorkflowsWithNodesIncluded(nodeTypes);
		workflowsWithPackageNodes.value = response?.data ?? [];
	}

	setIsVerifiedLatestPackage();
	setPackageVersion();
});
</script>

<template>
	<Modal
		width="640px"
		:name="COMMUNITY_PACKAGE_CONFIRM_MODAL_KEY"
		:title="getModalContent.title"
		:event-bus="modalBus"
		:center="true"
		:show-close="!loading"
		:before-close="onModalClose"
	>
		<template #content>
			<N8nText color="text-dark" :bold="true">{{ getModalContent.message }}</N8nText>
			<n8n-notice
				v-if="!isLatestPackageVerified"
				data-test-id="communityPackageManageConfirmModal-warning"
				:content="getModalContent.warning"
			/>
			<div :class="$style.descriptionContainer">
				<N8nText size="medium" color="text-base">
					{{ getModalContent.description }}
				</N8nText>
			</div>

			<NodesInWorkflowTable
				v-if="workflowsWithPackageNodes?.length"
				:data="workflowsWithPackageNodes"
			/>
		</template>
		<template #footer>
			<n8n-button
				:label="i18n.baseText('settings.communityNodes.confirmModal.cancel')"
				size="large"
				float="left"
				type="secondary"
				data-test-id="close-button"
				@click="onClick"
			/>
			<n8n-button
				:loading="loading"
				:disabled="loading"
				:label="loading ? getModalContent.buttonLoadingLabel : getModalContent.buttonLabel"
				size="large"
				float="right"
				@click="onConfirmButtonClick"
			/>
		</template>
	</Modal>
</template>

<style module lang="scss">
.descriptionContainer {
	display: flex;
	margin: var(--spacing-s) 0;
	flex-direction: column;
}

.descriptionIcon {
	align-self: center;
	color: var(--color-text-lighter);
}

.descriptionText {
	padding: 0 var(--spacing-xs);
}
</style>

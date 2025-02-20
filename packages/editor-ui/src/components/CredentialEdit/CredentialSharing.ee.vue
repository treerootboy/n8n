<template>
	<div :class="$style.container">
		<n8n-info-tip :bold="false" class="mb-s">
			<template v-if="credentialPermissions.isOwner">
				{{ $locale.baseText('credentialEdit.credentialSharing.info.owner') }}
			</template>
			<template v-else>
				{{ $locale.baseText('credentialEdit.credentialSharing.info.sharee', { interpolate: { credentialOwnerName } }) }}
			</template>
		</n8n-info-tip>
		<n8n-info-tip :bold="false" v-if="!credentialPermissions.isOwner && credentialPermissions.isInstanceOwner">
			{{ $locale.baseText('credentialEdit.credentialSharing.info.instanceOwner') }}
		</n8n-info-tip>
		<n8n-user-select
			v-if="credentialPermissions.updateSharing"
			size="large"
			:users="usersList"
			:currentUserId="usersStore.currentUser.id"
			:placeholder="$locale.baseText('credentialEdit.credentialSharing.select.placeholder')"
			@input="onAddSharee"
		>
			<template #prefix>
				<n8n-icon icon="search" />
			</template>
		</n8n-user-select>
		<n8n-users-list
			:users="sharedWithList"
			:currentUserId="usersStore.currentUser.id"
			:delete-label="$locale.baseText('credentialEdit.credentialSharing.list.delete')"
			:readonly="!credentialPermissions.updateSharing"
			@delete="onRemoveSharee"
		/>
	</div>
</template>

<script lang="ts">
import {IUser} from "@/Interface";
import mixins from "vue-typed-mixins";
import {showMessage} from "@/components/mixins/showMessage";
import { mapStores } from 'pinia';
import { useUsersStore } from '@/stores/users';

export default mixins(
	showMessage,
).extend({
	name: 'CredentialSharing',
	props: ['credential', 'credentialId', 'credentialData', 'sharedWith', 'credentialPermissions'],
	computed: {
		...mapStores(useUsersStore),
		usersList(): IUser[] {
			return this.usersStore.allUsers.filter((user: IUser) => {
				const isCurrentUser = user.id === this.usersStore.currentUser?.id;
				const isAlreadySharedWithUser = (this.credentialData.sharedWith || []).find((sharee: IUser) => sharee.id === user.id);

				return !isCurrentUser && !isAlreadySharedWithUser;
			});
		},
		sharedWithList(): IUser[] {
			return [
				{
					...(this.credential ? this.credential.ownedBy : this.usersStore.currentUser),
					isOwner: true,
				},
			].concat(this.credentialData.sharedWith || []);
		},
		credentialOwnerName(): string {
			return this.$store.getters['credentials/getCredentialOwnerName'](this.credentialId);
		},
	},
	methods: {
		async onAddSharee(userId: string) {
			const sharee =  this.usersStore.getUserById(userId);
			this.$emit('change', (this.credentialData.sharedWith || []).concat(sharee));
		},
		async onRemoveSharee(userId: string) {
			const user = this.usersStore.getUserById(userId);

			if (user) {
				const confirm = await this.confirmMessage(
					this.$locale.baseText('credentialEdit.credentialSharing.list.delete.confirm.message', { interpolate: { name: user.fullName || '' } }),
					this.$locale.baseText('credentialEdit.credentialSharing.list.delete.confirm.title'),
					null,
					this.$locale.baseText('credentialEdit.credentialSharing.list.delete.confirm.confirmButtonText'),
					this.$locale.baseText('credentialEdit.credentialSharing.list.delete.confirm.cancelButtonText'),
				);

				if (confirm) {
					this.$emit('change', this.credentialData.sharedWith.filter((sharee: IUser) => {
						return sharee.id !== user.id;
					}));
				}
			}
		},
		async loadUsers() {
			await this.usersStore.fetchUsers();
		},
	},
	mounted() {
		this.loadUsers();
	},
});
</script>

<style lang="scss" module>
.container {
	width: 100%;
	> * {
		margin-bottom: var(--spacing-l);
	}
}
</style>

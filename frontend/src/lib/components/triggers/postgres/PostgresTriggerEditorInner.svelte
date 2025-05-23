<script lang="ts">
	import { Alert, Button, TabContent } from '$lib/components/common'
	import Drawer from '$lib/components/common/drawer/Drawer.svelte'
	import DrawerContent from '$lib/components/common/drawer/DrawerContent.svelte'
	import Path from '$lib/components/Path.svelte'
	import Required from '$lib/components/Required.svelte'
	import ScriptPicker from '$lib/components/ScriptPicker.svelte'
	import { PostgresTriggerService, type Language, type Relations } from '$lib/gen'
	import { usedTriggerKinds, userStore, workspaceStore } from '$lib/stores'
	import { canWrite, emptyString, emptyStringTrimmed, sendUserToast } from '$lib/utils'
	import { createEventDispatcher } from 'svelte'
	import Section from '$lib/components/Section.svelte'
	import { Loader2, Save, X } from 'lucide-svelte'
	import Label from '$lib/components/Label.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import ResourcePicker from '$lib/components/ResourcePicker.svelte'
	import Tooltip from '$lib/components/Tooltip.svelte'
	import ToggleButton from '$lib/components/common/toggleButton-v2/ToggleButton.svelte'
	import ToggleButtonGroup from '$lib/components/common/toggleButton-v2/ToggleButtonGroup.svelte'
	import MultiSelect from 'svelte-multiselect'
	import PublicationPicker from './PublicationPicker.svelte'
	import SlotPicker from './SlotPicker.svelte'
	import { random_adj } from '$lib/components/random_positive_adjetive'
	import Tabs from '$lib/components/common/tabs/Tabs.svelte'
	import Tab from '$lib/components/common/tabs/Tab.svelte'
	import RelationPicker from './RelationPicker.svelte'
	import { invalidRelations } from './utils'
	import CheckPostgresRequirement from './CheckPostgresRequirement.svelte'
	import { base } from '$lib/base'

	let drawer: Drawer
	let is_flow: boolean = false
	let initialPath = ''
	let edit = true
	let itemKind: 'flow' | 'script' = 'script'
	let script_path = ''
	let initialScriptPath = ''
	let fixedScriptPath = ''
	let path: string = ''
	let pathError = ''
	let enabled = false
	let dirtyPath = false
	let can_write = true
	let drawerLoading = true
	let postgres_resource_path = ''
	let publication_name: string = ''
	let replication_slot_name: string = ''
	let relations: Relations[] | undefined = []
	let transaction_to_track: string[] = []
	let language: Language = 'Typescript'
	let loading = false
	type actions = 'create' | 'get'
	let selectedPublicationAction: actions
	let selectedSlotAction: actions
	let publicationItems: string[] = []
	let transactionType: string[] = ['Insert', 'Update', 'Delete']
	let tab: 'advanced' | 'basic' = 'basic'
	let isLoading = false
	async function createPublication() {
		try {
			const message = await PostgresTriggerService.createPostgresPublication({
				path: postgres_resource_path,
				publication: publication_name as string,
				workspace: $workspaceStore!,
				requestBody: {
					transaction_to_track: transaction_to_track,
					table_to_track: relations
				}
			})

			sendUserToast(message)
		} catch (error) {
			sendUserToast(error.body, true)
		}
	}

	async function createSlot() {
		try {
			const message = await PostgresTriggerService.createPostgresReplicationSlot({
				path: postgres_resource_path,
				workspace: $workspaceStore!,
				requestBody: {
					name: replication_slot_name
				}
			})
			sendUserToast(message)
		} catch (error) {
			sendUserToast(error.body, true)
		}
	}

	const dispatch = createEventDispatcher()

	$: is_flow = itemKind === 'flow'

	export async function openEdit(ePath: string, isFlow: boolean) {
		drawerLoading = true
		try {
			drawer?.openDrawer()
			initialPath = ePath
			itemKind = isFlow ? 'flow' : 'script'
			edit = true
			dirtyPath = false
			selectedPublicationAction = 'get'
			selectedSlotAction = 'get'
			selectedPublicationAction = selectedPublicationAction
			selectedSlotAction = selectedSlotAction
			relations = []
			transaction_to_track = []
			tab = 'basic'
			await loadTrigger()
		} catch (err) {
			sendUserToast(`Could not load postgres trigger: ${err.body}`, true)
		} finally {
			drawerLoading = false
		}
	}

	export async function openNew(
		nis_flow: boolean,
		fixedScriptPath_?: string,
		defaultValues?: Record<string, any>
	) {
		drawerLoading = true
		try {
			selectedPublicationAction = 'create'
			selectedSlotAction = 'create'
			tab = 'basic'

			drawer?.openDrawer()
			is_flow = nis_flow
			itemKind = nis_flow ? 'flow' : 'script'
			initialScriptPath = ''
			fixedScriptPath = fixedScriptPath_ ?? ''
			script_path = fixedScriptPath
			path = ''
			initialPath = ''
			postgres_resource_path = defaultValues?.postgres_resource_path ?? ''
			edit = false
			dirtyPath = false
			publication_name = `windmill_publication_${random_adj()}`
			replication_slot_name = `windmill_replication_${random_adj()}`
			transaction_to_track = defaultValues?.publication.transaction_to_track || [
				'Insert',
				'Update',
				'Delete'
			]
			relations = defaultValues?.publication.table_to_track || [
				{
					schema_name: 'public',
					table_to_track: []
				}
			]
		} finally {
			drawerLoading = false
		}
	}

	async function loadTrigger(): Promise<void> {
		const s = await PostgresTriggerService.getPostgresTrigger({
			workspace: $workspaceStore!,
			path: initialPath
		})
		script_path = s.script_path
		initialScriptPath = s.script_path

		is_flow = s.is_flow
		path = s.path
		enabled = s.enabled
		postgres_resource_path = s.postgres_resource_path
		publication_name = s.publication_name
		replication_slot_name = s.replication_slot_name

		const publication_data = await PostgresTriggerService.getPostgresPublication({
			path: postgres_resource_path,
			workspace: $workspaceStore!,
			publication: publication_name
		})
		transaction_to_track = [...publication_data.transaction_to_track]
		relations = publication_data.table_to_track ?? []
		can_write = canWrite(s.path, s.extra_perms, $userStore)
	}

	async function updateTrigger(): Promise<void> {
		if (
			relations &&
			invalidRelations(relations, {
				showError: true,
				trackSchemaTableError: true
			}) === true
		) {
			return
		}
		isLoading = true
		try {
			if (edit) {
				await PostgresTriggerService.updatePostgresTrigger({
					workspace: $workspaceStore!,
					path: initialPath,
					requestBody: {
						path,
						script_path,
						is_flow,
						postgres_resource_path,
						enabled,
						replication_slot_name,
						publication_name,
						publication:
							tab === 'basic'
								? {
										transaction_to_track,
										table_to_track: relations
									}
								: undefined
					}
				})
				sendUserToast(`PostgresTrigger ${path} updated`)
			} else {
				await PostgresTriggerService.createPostgresTrigger({
					workspace: $workspaceStore!,
					requestBody: {
						path,
						script_path,
						is_flow,
						enabled: true,
						postgres_resource_path,
						replication_slot_name: tab === 'basic' ? undefined : replication_slot_name,
						publication_name: tab === 'basic' ? undefined : publication_name,
						publication: {
							transaction_to_track,
							table_to_track: relations
						}
					}
				})
				sendUserToast(`PostgresTrigger ${path} created`)
			}
			isLoading = false
			if (!$usedTriggerKinds.includes('postgres')) {
				$usedTriggerKinds = [...$usedTriggerKinds, 'postgres']
			}
			dispatch('update')
			drawer.closeDrawer()
		} catch (error) {
			isLoading = false
			sendUserToast(error.body || error.message, true)
		}
	}

	const getTemplateScript = async () => {
		if (!relations || relations.length === 0 || emptyString(postgres_resource_path)) {
			sendUserToast('You must pick a database resource and choose at least one schema', true)
			return
		}
		try {
			loading = true
			let templateId = await PostgresTriggerService.createTemplateScript({
				workspace: $workspaceStore!,
				requestBody: {
					relations,
					language,
					postgres_resource_path
				}
			})
			window.open(`${base}/scripts/add?id=${templateId}`)
			loading = false
		} catch (error) {
			loading = false
			sendUserToast(error.body, true)
		}
	}
</script>

<Drawer size="800px" bind:this={drawer}>
	<DrawerContent
		title={edit
			? can_write
				? `Edit Postgres trigger ${initialPath}`
				: `Postgres trigger ${initialPath}`
			: 'New Postgres trigger'}
		on:close={drawer.closeDrawer}
	>
		<svelte:fragment slot="actions">
			{#if !drawerLoading && can_write}
				{#if edit}
					<div class="mr-8 center-center -mt-1">
						<Toggle
							disabled={!can_write}
							checked={enabled}
							options={{ right: 'enable', left: 'disable' }}
							on:change={async (e) => {
								await PostgresTriggerService.setPostgresTriggerEnabled({
									path: initialPath,
									workspace: $workspaceStore ?? '',
									requestBody: { enabled: e.detail }
								})
								sendUserToast(
									`${e.detail ? 'enabled' : 'disabled'} postgres trigger ${initialPath}`
								)
							}}
						/>
					</div>
				{/if}
				<Button
					startIcon={{ icon: Save }}
					disabled={pathError != '' ||
						emptyString(postgres_resource_path) ||
						emptyString(script_path) ||
						(tab === 'advanced' && emptyString(replication_slot_name)) ||
						emptyString(publication_name) ||
						(relations && tab === 'basic' && relations.length === 0) ||
						transaction_to_track.length === 0 ||
						!can_write}
					on:click={updateTrigger}
					loading={isLoading}
				>
					Save
				</Button>
			{/if}
		</svelte:fragment>
		{#if drawerLoading}
			<div class="flex flex-col items-center justify-center h-full w-full">
				<Loader2 size="50" class="animate-spin" />
				<p>Loading...</p>
			</div>
		{:else}
			<Alert title="Info" type="info">
				{#if edit}
					Changes can take up to 30 seconds to take effect.
				{:else}
					New postgres triggers can take up to 30 seconds to start listening.
				{/if}
			</Alert>
			<div class="flex flex-col gap-12 mt-6">
				<Label label="Path">
					<Path
						bind:dirty={dirtyPath}
						bind:error={pathError}
						bind:path
						{initialPath}
						checkInitialPathExistence={!edit}
						namePlaceholder="postgres_trigger"
						kind="postgres_trigger"
						disabled={!can_write}
					/>
				</Label>
				<Section label="Runnable">
					<p class="text-xs text-tertiary">
						Pick a script or flow to be triggered <Required required={true} />
					</p>
					<div class="flex flex-row mb-2">
						<ScriptPicker
							disabled={fixedScriptPath != '' || !can_write}
							initialPath={fixedScriptPath || initialScriptPath}
							kinds={['script']}
							allowFlow={true}
							bind:itemKind
							bind:scriptPath={script_path}
							allowRefresh
						/>

						{#if emptyStringTrimmed(script_path) && is_flow === false}
							<div class="flex">
								<Button
									disabled={!can_write}
									btnClasses="ml-4 mt-2"
									color="dark"
									size="xs"
									on:click={getTemplateScript}
									target="_blank"
									{loading}
									>Create from template
									<Tooltip light>
										The conversion requires a <strong>database resource</strong> and at least one
										<strong>schema</strong>
										to be set.<br />
										Please ensure these conditions are met before proceeding.
									</Tooltip>
								</Button>
							</div>
						{/if}
					</div>
				</Section>
				<Section label="Database">
					<p class="text-xs text-tertiary mb-2">
						Pick a database to connect to <Required required={true} />
					</p>
					<div class="flex flex-col gap-8">
						<div class="flex flex-col gap-2">
							<ResourcePicker
								disabled={!can_write}
								bind:value={postgres_resource_path}
								resourceType={'postgresql'}
							/>
							<CheckPostgresRequirement bind:postgres_resource_path bind:can_write />
						</div>

						{#if postgres_resource_path}
							<Label label="Transactions">
								<svelte:fragment slot="header">
									<Tooltip>
										<p>
											Choose the types of database transactions that should trigger a script or
											flow. You can select from <strong>Insert</strong>, <strong>Update</strong>,
											<strong>Delete</strong>, or any combination of these operations to define when
											the trigger should activate.
										</p>
									</Tooltip>
								</svelte:fragment>
								<MultiSelect
									noMatchingOptionsMsg=""
									createOptionMsg={null}
									duplicates={false}
									options={transactionType}
									allowUserOptions="append"
									bind:selected={transaction_to_track}
									ulOptionsClass={'!bg-surface !text-sm'}
									ulSelectedClass="!text-sm"
									outerDivClass="!bg-surface !min-h-[38px] !border-[#d1d5db]"
									placeholder="Select transactions"
									--sms-options-margin="4px"
									--sms-open-z-index="100"
								>
									<svelte:fragment slot="remove-icon">
										<div class="hover:text-primary p-0.5">
											<X size={12} />
										</div>
									</svelte:fragment>
								</MultiSelect>
							</Label>
							<Label label="Table Tracking">
								<svelte:fragment slot="header">
									<Tooltip
										documentationLink="https://www.windmill.dev/docs/core_concepts/postgres_triggers#define-what-to-track"
									>
										<p>
											Select the tables to track. You can choose to track
											<strong>all tables in your database</strong>,
											<strong>all tables within a specific schema</strong>,
											<strong>specific tables in a schema</strong>, or even
											<strong>specific columns of a table</strong>. Additionally, you can apply a
											<strong>filter</strong> to retrieve only rows that do not match the specified criteria.
										</p>
									</Tooltip>
								</svelte:fragment>
								<Tabs bind:selected={tab}>
									<Tab value="basic"
										><div class="flex flex-row gap-1"
											>Basic<Tooltip
												documentationLink="https://www.windmill.dev/docs/core_concepts/postgres_triggers#define-what-to-track"
												><p
													>Choose the <strong>relations</strong> to track without worrying about the
													underlying mechanics of creating a
													<strong>publication</strong>
													or <strong>slot</strong>. This simplified option lets you focus only on
													the data you want to monitor.</p
												></Tooltip
											></div
										></Tab
									>
									<Tab value="advanced"
										><div class="flex flex-row gap-1"
											>Advanced<Tooltip
												documentationLink="https://www.windmill.dev/docs/core_concepts/postgres_triggers#advanced"
												><p
													>Select a specific <strong>publication</strong> from your database to
													track, and manage it by <strong>creating</strong>,
													<strong>updating</strong>, or <strong>deleting</strong>. For
													<strong>slots</strong>, you can <strong>create</strong> or
													<strong>delete</strong>
													them. Both <strong>non-active slots</strong> and the
													<strong>currently used slot</strong> by the trigger will be retrieved from
													your database for management.</p
												></Tooltip
											></div
										></Tab
									>
									<svelte:fragment slot="content">
										<div class="mt-5 overflow-hidden bg-surface">
											<TabContent value="basic">
												<RelationPicker {can_write} bind:relations />
											</TabContent>
											<TabContent value="advanced">
												<div class="flex flex-col gap-6"
													><Section
														small
														label="Replication slot"
														tooltip="Choose and manage the slots for your trigger. You can create or delete slots. Both non-active slots and the currently used slot by the trigger (if any) will be retrieved from your database for management."
														documentationLink="https://www.windmill.dev/docs/core_concepts/postgres_triggers#managing-postgres-replication-slots"
													>
														<div class="flex flex-col gap-3">
															<ToggleButtonGroup
																bind:selected={selectedSlotAction}
																on:selected={() => {
																	replication_slot_name = ''
																}}
																disabled={!can_write}
																let:item
															>
																<ToggleButton value="create" label="Create Slot" {item} />
																<ToggleButton value="get" label="Get Slot" {item} />
															</ToggleButtonGroup>
															{#if selectedSlotAction === 'create'}
																<div class="flex gap-3">
																	<input
																		type="text"
																		bind:value={replication_slot_name}
																		placeholder={'Choose a slot name'}
																	/>
																	<Button
																		color="light"
																		size="xs"
																		variant="border"
																		disabled={emptyStringTrimmed(replication_slot_name)}
																		on:click={createSlot}>Create</Button
																	>
																</div>
															{:else}
																<SlotPicker
																	bind:edit
																	{postgres_resource_path}
																	bind:replication_slot_name
																/>
															{/if}
														</div>
													</Section>

													<Section
														small
														label="Publication"
														tooltip="Select and manage the publications for tracking data. You can create, update, or delete publications. Only existing publications in your database will be available for selection, giving you full control over what data is tracked."
														documentationLink="https://www.windmill.dev/docs/core_concepts/postgres_triggers#managing-postgres-publications"
													>
														<div class="flex flex-col gap-3">
															<ToggleButtonGroup
																selected={selectedPublicationAction}
																disabled={!can_write}
																on:selected={({ detail }) => {
																	selectedPublicationAction = detail
																	if (selectedPublicationAction === 'create') {
																		publication_name = `windmill_publication_${random_adj()}`
																		relations = [{ schema_name: 'public', table_to_track: [] }]
																		return
																	}

																	publication_name = ''
																	relations = []
																	transaction_to_track = []
																}}
																let:item
															>
																<ToggleButton value="create" label="Create Publication" {item} />
																<ToggleButton value="get" label="Get Publication" {item} />
															</ToggleButtonGroup>
															{#if selectedPublicationAction === 'create'}
																<div class="flex gap-3">
																	<input
																		disabled={!can_write}
																		type="text"
																		bind:value={publication_name}
																		placeholder={'Publication Name'}
																	/>
																	<Button
																		color="light"
																		size="xs"
																		variant="border"
																		disabled={emptyStringTrimmed(publication_name) ||
																			(relations && relations.length === 0) ||
																			!can_write}
																		on:click={createPublication}>Create</Button
																	>
																</div>
															{:else}
																<PublicationPicker
																	{can_write}
																	{postgres_resource_path}
																	bind:transaction_to_track
																	bind:relations
																	bind:items={publicationItems}
																	bind:publication_name
																/>
															{/if}
															<RelationPicker {can_write} bind:relations />
														</div>
													</Section></div
												>
											</TabContent>
										</div>
									</svelte:fragment>
								</Tabs>
							</Label>
						{/if}
					</div>
				</Section>
			</div>
		{/if}
	</DrawerContent>
</Drawer>

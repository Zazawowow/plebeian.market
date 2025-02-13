<script lang="ts">
	import type { Selected } from 'bits-ui'
	import { goto } from '$app/navigation'
	import { processAppSettings, submitAppSettings } from '$lib/appSettings'
	import { Accordion } from '$lib/components/ui/accordion'
	import AccordionContent from '$lib/components/ui/accordion/accordion-content.svelte'
	import AccordionItem from '$lib/components/ui/accordion/accordion-item.svelte'
	import AccordionTrigger from '$lib/components/ui/accordion/accordion-trigger.svelte'
	import Button from '$lib/components/ui/button/button.svelte'
	import Checkbox from '$lib/components/ui/checkbox/checkbox.svelte'
	import * as Command from '$lib/components/ui/command/index.js'
	import { Input } from '$lib/components/ui/input'
	import { Label } from '$lib/components/ui/label'
	import * as Popover from '$lib/components/ui/popover/index.js'
	import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '$lib/components/ui/select'
	import Separator from '$lib/components/ui/separator/separator.svelte'
	import { availabeLogos } from '$lib/constants'
	import { copyToClipboard } from '$lib/utils'
	import { generateSecretKey, getPublicKey, nip19 } from 'nostr-tools'
	import { npubEncode } from 'nostr-tools/nip19'
	import { onMount, tick } from 'svelte'
	import { toast } from 'svelte-sonner'

	import type { PageData } from './$types'

	export let data: PageData
	const { currencies, appSettings, adminUsers, instancePass } = data
	let checked = true
	let selectedCurrency: Selected<string> = { value: currencies[0], label: currencies[0] }
	let manualNsec = false
	let newInstanceNsec = ''
	let newInstanceNpub = ''
	let adminsList: string[] = adminUsers.map((user) => npubEncode(user))
	let inputValue: string = ''

	let logoUrl: string = ''
	let open = false

	function setGeneratedSk() {
		manualNsec = false
		const sk = generateSecretKey()
		const newPk = getPublicKey(sk)

		newInstanceNsec = nip19.nsecEncode(sk)
		newInstanceNpub = nip19.npubEncode(newPk)
	}

	function handleNpubNsecInput(event: Event) {
		const inputValue = (event.target as HTMLInputElement).value

		if (inputValue.startsWith('nsec')) {
			try {
				const { type, data } = nip19.decode(inputValue)
				if (type === 'nsec') {
					manualNsec = true
					newInstanceNsec = inputValue
					const newPk = getPublicKey(data as Uint8Array)
					newInstanceNpub = nip19.npubEncode(newPk)
				} else {
					throw new Error('Invalid nsec')
				}
			} catch (error) {
				console.error('Invalid nsec:', error)
				toast.error('Invalid nsec provided')
			}
		} else {
			manualNsec = false
		}
	}

	async function handleSubmit(event: SubmitEvent) {
		event.preventDefault()
		const formData = new FormData(event.currentTarget as HTMLFormElement)
		const formObject = Object.fromEntries(formData)

		try {
			const processedData = processAppSettings(
				{
					...formObject,
					instancePk: newInstanceNpub,
					instanceSk: newInstanceNsec,
					allowRegister: checked,
					defaultCurrency: selectedCurrency.value,
					logoUrl: logoUrl || undefined,
					contactEmail: formObject.contactEmail || undefined,
				},
				true,
				instancePass,
			)
			await submitAppSettings(processedData, true)
			toast.success('App settings successfully updated!')
			goto('/', { invalidateAll: true })
		} catch (e) {
			console.error('Failed to submit form', e)
			if (e instanceof Error && e.message) {
				try {
					const errors = JSON.parse(e.message)
					if (Array.isArray(errors)) {
						errors.forEach((error) => toast.error(error.message))
					} else {
						toast.error(e.message)
					}
				} catch {
					toast.error(e.message)
				}
			} else {
				toast.error('An unknown error occurred')
			}
		}
	}
	function closeAndFocusTrigger(triggerId: string) {
		open = false
		tick().then(() => {
			document.getElementById(triggerId)?.focus()
		})
	}

	onMount(() => {
		if (!appSettings?.isFirstTimeRunning) {
			goto('/', { invalidateAll: true })
		}
	})
</script>

<div class="px-4 py-10 lg:px-12">
	<div class="mx-auto max-w-2xl flex flex-col gap-2">
		<main class="text-black">
			<div class="px-4 lg:px-12">
				<div class="container">
					<h2 class="max-w-2xl">Instance setup</h2>
					<Separator class=" my-2" />
					<form on:submit|preventDefault={handleSubmit} class="max-w-2xl flex flex-col gap-3">
						<h3>Identity</h3>
						<Label class="truncate font-bold">Instance npub</Label>
						<div class="flex items-center gap-2">
							<Input
								bind:value={newInstanceNpub}
								class="border-black border-2"
								name="instancePk"
								placeholder="instance npub"
								type="text"
								readonly
							/>
							<Button variant="tertiary" on:click={setGeneratedSk}>Generate</Button>
						</div>

						<Accordion>
							<AccordionItem value="advanced-options">
								<AccordionTrigger>Advanced Options</AccordionTrigger>
								<AccordionContent>
									<Label class="truncate font-bold">Instance nsec</Label>
									<Input
										class="border-black border-2"
										placeholder="Paste your nsec here"
										name="customInstanceSk"
										type="text"
										on:input={handleNpubNsecInput}
										bind:value={newInstanceNsec}
									/>
								</AccordionContent>
							</AccordionItem>
						</Accordion>

						{#if newInstanceNsec}
							<Label class="truncate font-bold">New nsec</Label>
							<div class="flex flex-row gap-2">
								<Input class="border-black border-2" value={newInstanceNsec} readonly name="instanceSk" />
								<Button variant="secondary" on:click={() => copyToClipboard(newInstanceNsec)}>
									<span class="i-mingcute-clipboard-fill text-black w-6 h-6"></span>
								</Button>
							</div>
						{/if}

						<Label class="truncate font-bold">Owner npub</Label>
						<div class="flex flex-row gap-2">
							<Input class=" border-black border-2" name="ownerPk" placeholder="owner npub" type="text" id="ownerPkInput" />
							<Button
								variant="secondary"
								on:click={async () => {
									const user = await window.nostr?.getPublicKey()
									const inputElement = document.getElementById('ownerPkInput')
									if (inputElement && 'value' in inputElement) {
										inputElement.value = npubEncode(user)
									}
								}}
							>
								<span class=" i-mingcute-key-2-fill text-black w-6 h-6"></span>
							</Button>
						</div>
						<div class=" flex-grow">
							<Label class="truncate font-bold required-mark">Instance name</Label>
							<Input required class="border-black border-2" name="instanceName" placeholder="instance name" type="text" />
						</div>

						<div class=" flex flex-col gap-2">
							<div>
								<Label class="truncate font-bold">Logo url</Label>
								<Popover.Root bind:open let:ids>
									<Popover.Trigger asChild let:builder>
										<Button
											builders={[builder]}
											variant="outline"
											role="combobox"
											aria-expanded={open}
											class="w-full justify-between border-black border-2"
										>
											{#if logoUrl}
												{availabeLogos.find((logo) => logo.value === logoUrl)?.label || logoUrl}
											{:else}
												<span class=" opacity-50">Select logo</span>
											{/if}
										</Button>
									</Popover.Trigger>
									<Popover.Content class=" p-0">
										<Command.Root>
											<Command.Input placeholder="Select logo or introduce image url..." bind:value={logoUrl} />
											<Command.Empty>No logo found.</Command.Empty>
											<Command.Group>
												{#each availabeLogos as logo}
													<Command.Item
														value={logo.value}
														onSelect={(currentValue) => {
															logoUrl = currentValue
															closeAndFocusTrigger(ids.trigger)
														}}
													>
														+
														{logo.label}
													</Command.Item>
												{/each}
											</Command.Group>
										</Command.Root>
									</Popover.Content>
								</Popover.Root>
							</div>
							<div class="self-center">
								{#if logoUrl}
									<img
										class=" max-w-28"
										src={logoUrl}
										alt="logo preview"
										on:error={(e) => {
											if (e.target instanceof HTMLImageElement) {
												e.target.src = availabeLogos[0].value
											}
										}}
									/>
								{/if}
							</div>
						</div>
						<Label class="truncate font-bold">Contact email</Label>
						<Input class="border-black border-2" name="contactEmail" placeholder="contact email" type="email" />
						<Separator class=" my-2" />
						<h3>Crew</h3>
						{#each adminsList as admin}
							<div class=" grid grid-cols-[1fr_auto] items-center">
								<span class="truncate">{admin}</span>
								<Button
									type="button"
									size="icon"
									variant="outline"
									class=" bg-red-500"
									on:click={() => (adminsList = adminsList.filter((value) => value !== admin))}
								>
									<span class="i-mdi-trash-can"></span>
								</Button>
							</div>
						{/each}
						<textarea name="adminsList" value={adminsList} hidden />
						<Input type="text" bind:value={inputValue} />
						<Button
							type="button"
							on:click={() => {
								inputValue = inputValue.trim()
								if (inputValue) {
									adminsList = [...adminsList, inputValue]
									inputValue = ''
								}
							}}
						>
							Add Admin
						</Button>

						<Separator class=" my-2" />
						<h3>Miscellanea</h3>
						<div class="flex flex-col gap-4">
							<div>
								<Label class="truncate font-bold">Default currency</Label>
								<Select bind:selected={selectedCurrency} name="defaultCurrency">
									<SelectTrigger class="border-black border-2">
										<SelectValue placeholder="Currency" />
									</SelectTrigger>
									<SelectContent class="border-black border-2 max-h-[350px] overflow-y-auto">
										{#each currencies as currency}
											<SelectItem value={currency}>{currency}</SelectItem>
										{/each}
									</SelectContent>
								</Select>
							</div>
							<div class=" inline-flex gap-3">
								<Label class="truncate font-bold required-mark">Allow register</Label>
								<Checkbox required class="border-black border-2" name="allowRegister" bind:checked placeholder="allow register" />
							</div>
						</div>

						<Separator class="max-w-2xl my-8" />

						<Button variant="primary" type="submit" class="w-full">Submit</Button>
					</form>
				</div>
			</div>
		</main>
	</div>
</div>

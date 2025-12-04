<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import {
		Card,
		CardContent,
		CardDescription,
		CardHeader,
		CardTitle
	} from '$lib/components/ui/card';
	import { Input } from '$lib/components/ui/input';
	import { Label } from '$lib/components/ui/label';
	import { Textarea } from '$lib/components/ui/textarea';
	import { toast } from 'svelte-sonner';
	import * as NativeSelect from '$lib/components/ui/native-select';
	import { Loader2, Send, Search, Plus, Trash2, Save, X } from 'lucide-svelte';
	import { env } from '$env/dynamic/public';
	import { contractAddresses } from '$lib/contracts';
	import { onMount } from 'svelte';

	type Transaction = {
		to: string;
		data: string;
		value: string;
		count: number;
		selectedContract: string;
	};

	let apiUrl = $state(env.PUBLIC_API_BASE_URL || 'http://localhost:3000');
	let apiKey = $state(env.PUBLIC_API_KEY || '');
	let privateKey = $state('');
	let rpc = $state('https://rpc.linea.build');
	let transactions = $state<Transaction[]>([
		{ to: '', data: '0x', value: '0', count: 1, selectedContract: contractAddresses[0].address }
	]);
	type JobStatus = {
		id?: string;
		status?: string;
		wallet?: string;
		balances?: {
			eth?: string;
			token?: string;
		};
		planned?: {
			count?: number;
			totalAmount?: string;
		};
		completed?: number;
		transactions?: Array<{
			hash?: string;
			status?: number;
			error?: string;
		}>;
		createdAt?: number;
		error?: string;
	};

	let jobId = $state('');
	let isSubmitting = $state(false);
	let isCheckingStatus = $state(false);
	let jobStatus = $state<JobStatus | null>(null);

	const STORAGE_KEY = 'l20ui_transactions';

	onMount(() => {
		loadTransactions();
	});

	function saveTransactions() {
		try {
			const dataToSave = {
				apiUrl,
				apiKey,
				rpc,
				transactions
			};
			localStorage.setItem(STORAGE_KEY, JSON.stringify(dataToSave));
			toast.success('Transactions saved successfully!');
		} catch (error) {
			toast.error(`Failed to save: ${error}`);
		}
	}

	function loadTransactions() {
		try {
			const saved = localStorage.getItem(STORAGE_KEY);
			if (saved) {
				const data = JSON.parse(saved);
				if (data.apiUrl) apiUrl = data.apiUrl;
				if (data.apiKey) apiKey = data.apiKey;
				if (data.rpc) rpc = data.rpc;
				if (data.transactions && Array.isArray(data.transactions)) {
					transactions = data.transactions;
				}
				toast.success('Transactions loaded from storage');
			}
		} catch (error) {
			console.error('Failed to load transactions:', error);
		}
	}

	function clearLocalStorage() {
		try {
			localStorage.removeItem(STORAGE_KEY);
			toast.success('Local storage cleared successfully!');
		} catch (error) {
			toast.error(`Failed to clear storage: ${error}`);
		}
	}

	function addTransaction() {
		transactions = [
			...transactions,
			{ to: '', data: '0x', value: '0', count: 1, selectedContract: contractAddresses[0].address }
		];
	}

	function removeTransaction(index: number) {
		transactions = transactions.filter((_, i) => i !== index);
	}

	function handleContractSelect(index: number, address: string) {
		transactions[index].selectedContract = address;
		if (address !== '') {
			transactions[index].to = address;
		} else {
			// Clear the address when custom is selected
			transactions[index].to = '';
		}
	}

	async function submitBatchTransactions() {
		isSubmitting = true;

		try {
			// Validate transactions
			if (transactions.length === 0) {
				toast.error('Please add at least one transaction');
				return;
			}

			// Check if all transactions have required fields
			const invalidTx = transactions.findIndex((tx) => !tx.to.trim() || !tx.data.trim());
			if (invalidTx !== -1) {
				toast.error(`Transaction #${invalidTx + 1} is missing required fields (to, data)`);
				return;
			}

			if (!privateKey.trim()) {
				toast.error('Please enter your private key');
				return;
			}

			// Prepare transactions for API
			const txPayload = transactions.map((tx) => ({
				to: tx.to,
				data: tx.data,
				value: tx.value,
				count: tx.count
			}));

			const response = await fetch(`${apiUrl}/interact/batch-send-raw`, {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
					'x-api-key': apiKey
				},
				body: JSON.stringify({
					privateKey: privateKey,
					rpc: rpc,
					transactions: txPayload
				})
			});

			const data = await response.json();

			if (response.ok) {
				if (data.jobId) {
					jobId = data.jobId;
					toast.success(`Batch submitted successfully! Job ID: ${data.jobId}`);
				} else {
					toast.success(data.message || 'Batch submitted successfully!');
				}
				// Reset transactions to single empty one
				transactions = [
					{
						to: '',
						data: '0x',
						value: '0',
						count: 1,
						selectedContract: contractAddresses[0].address
					}
				];
			} else {
				toast.error(data.error || 'Failed to submit batch');
			}
		} catch (error) {
			toast.error(`Error: ${error}`);
		} finally {
			isSubmitting = false;
		}
	}

	async function checkJobStatus() {
		if (!jobId.trim()) return;

		isCheckingStatus = true;
		jobStatus = null;

		try {
			const response = await fetch(`${apiUrl}/batch/${jobId.trim()}`, {
				headers: {
					'x-api-key': apiKey
				}
			});
			const data = await response.json();

			if (response.ok) {
				jobStatus = data;
			} else {
				jobStatus = { error: data.error || 'Failed to fetch job status' };
			}
		} catch (error) {
			jobStatus = { error: `Error: ${error}` };
		} finally {
			isCheckingStatus = false;
		}
	}

	function getStatusColor(status: string) {
		switch (status?.toLowerCase()) {
			case 'completed':
				return 'text-green-600';
			case 'running':
			case 'processing':
			case 'queued':
				return 'text-blue-600';
			case 'failed':
				return 'text-red-600';
			default:
				return 'text-gray-600';
		}
	}
</script>

<div class="container mx-auto max-w-4xl p-6">
	<h1 class="mb-8 text-3xl font-bold">l20-ui</h1>

	<!-- API Configuration -->
	<Card class="mb-6">
		<CardHeader>
			<CardTitle>Configuration</CardTitle>
			<CardDescription>Configure your API connection settings</CardDescription>
		</CardHeader>
		<CardContent class="space-y-4">
			<div>
				<Label for="api-url">API URL</Label>
				<Input id="api-url" bind:value={apiUrl} placeholder="http://localhost:3000" class="mt-2" />
			</div>
			<div>
				<Label for="api-key">API Key</Label>
				<Input
					id="api-key"
					type="password"
					bind:value={apiKey}
					placeholder="your_secret_api_key"
					class="mt-2"
				/>
			</div>
			<div>
				<Label for="rpc">RPC URL</Label>
				<Input id="rpc" bind:value={rpc} placeholder="https://rpc.linea.build" class="mt-2" />
			</div>
		</CardContent>
	</Card>

	<!-- Submit Batch Transactions -->
	<Card class="mb-6">
		<CardHeader>
			<CardTitle>Batch Send Raw Transactions</CardTitle>
			<CardDescription>
				Build and send multiple transactions in batch. Each transaction requires a destination
				address, data, value, and repeat count.
			</CardDescription>
		</CardHeader>
		<CardContent class="space-y-4">
			<div>
				<Label for="private-key">Private Key</Label>
				<Input
					id="private-key"
					type="password"
					bind:value={privateKey}
					placeholder="0x..."
					class="mt-2 font-mono"
				/>
			</div>

			<!-- Transactions List -->
			<div class="space-y-4">
				<div class="flex items-center justify-between py-2 flex-wrap gap-2">
					<div>
						<Label class="text-base">Transactions</Label>
						<p class="text-xs text-muted-foreground mt-1">
							Build your transaction batch ({transactions.length} total)
						</p>
					</div>
					<div class="flex gap-2">
						<Button
							onclick={clearLocalStorage}
							size="sm"
							variant="outline"
							class="gap-2"
						>
							<X class="h-4 w-4" />
							Clear
						</Button>
						<Button onclick={saveTransactions} size="sm" variant="outline" class="gap-2">
							<Save class="h-4 w-4" />
							Save
						</Button>
						<Button onclick={addTransaction} size="sm" variant="outline" class="gap-2">
							<Plus class="h-4 w-4" />
							Add Transaction
						</Button>
					</div>
				</div>

				{#each transactions as tx, i (`tx-${i}-${tx.selectedContract}`)}
					<div class="rounded-lg border bg-card p-4 space-y-4 shadow-sm">
						<div class="flex items-center justify-between pb-3 border-b">
							<div class="flex items-center gap-2">
								<div
									class="flex h-8 w-8 items-center justify-center rounded-full bg-primary/10 text-primary text-sm font-semibold"
								>
									{i + 1}
								</div>
								<span class="font-semibold">Transaction</span>
							</div>
							{#if transactions.length > 1}
								<Button
									onclick={() => removeTransaction(i)}
									size="sm"
									variant="ghost"
									class="h-8 w-8 p-0 hover:bg-destructive/10"
								>
									<Trash2 class="h-4 w-4 text-destructive" />
								</Button>
							{/if}
						</div>

						<div class="space-y-3">
							<div class="space-y-2">
								<Label for={`contract-select-${i}`} class="text-sm font-medium">
									Select Contract
								</Label>
								<NativeSelect.Root
									id={`contract-select-${i}`}
									onchange={(e) => handleContractSelect(i, e.currentTarget.value)}
								>
									{#each contractAddresses as contract (contract.address || contract.name)}
										<NativeSelect.Option
											value={contract.address}
											selected={tx.selectedContract === contract.address}
										>
											{contract.name}
											{contract.address
												? `(${contract.address.slice(0, 6)}...${contract.address.slice(-4)})`
												: ''}
										</NativeSelect.Option>
									{/each}
								</NativeSelect.Root>
							</div>

							<div class="space-y-2">
								<Label for={`to-${i}`} class="text-sm font-medium">
									To Address
									<div class="flex items-center">
										<span class="text-xs text-muted-foreground ml-1">
										{tx.selectedContract === ''
											? '(Enter custom address)'
											: '(Auto-filled from contract)'}
										</span>
										<span class="ml-2 text-xs text-blue-600">
											{#if tx.selectedContract !== ''}
												<a
													href={
														contractAddresses.find(
															(c) => c.address === tx.selectedContract
														)?.link || '#'
													}
													target="_blank"
													rel="noopener noreferrer"
												>
													Visit Page
												</a>
											{/if}
										</span>
									</div>
								</Label>
								<div class="relative">
									<Input
										id={`to-${i}`}
										bind:value={tx.to}
										placeholder={tx.selectedContract === ''
											? '0xCONTRACT_ADDRESS'
											: 'Auto-filled from selected contract'}
										readonly={tx.selectedContract !== ''}
										class={`font-mono text-sm ${tx.selectedContract !== '' ? 'bg-muted cursor-not-allowed' : ''}`}
									/>
									{#if tx.selectedContract !== ''}
										<div
											class="absolute right-3 top-1/2 -translate-y-1/2 text-xs text-muted-foreground"
										>
											🔒
										</div>
									{/if}
								</div>
								{#if tx.to && tx.to.length > 0 && !tx.to.startsWith('0x')}
									<p class="text-xs text-destructive">Address should start with 0x</p>
								{:else if tx.selectedContract !== ''}
									<p class="text-xs text-muted-foreground">
										Select "Custom Address" to enter manually
									</p>
								{/if}
							</div>

							<div class="space-y-2">
								<Label for={`data-${i}`} class="text-sm font-medium">
									Data
									<span class="text-xs text-muted-foreground ml-1">(Hex)</span>
								</Label>
								<Textarea
									id={`data-${i}`}
									bind:value={tx.data}
									placeholder="0x4e71d92d"
									rows={3}
									class="font-mono text-sm resize-none"
								/>
							</div>

							<div class="grid grid-cols-1 md:grid-cols-2 gap-3">
								<div class="space-y-2">
									<Label for={`value-${i}`} class="text-sm font-medium">
										Value
										<span class="text-xs text-muted-foreground ml-1">(ETH)</span>
									</Label>
									<Input
										id={`value-${i}`}
										type="number"
										step="0.000001"
										bind:value={tx.value}
										placeholder="0.0"
										class="text-sm"
									/>
								</div>
								<div class="space-y-2">
									<Label for={`count-${i}`} class="text-sm font-medium">
										Count
										<span class="text-xs text-muted-foreground ml-1">(Repeats)</span>
									</Label>
									<Input
										id={`count-${i}`}
										type="number"
										min="1"
										step="1"
										bind:value={tx.count}
										placeholder="1"
										class="text-sm"
									/>
								</div>
							</div>
						</div>
					</div>
				{/each}
			</div>

			<Button
				onclick={submitBatchTransactions}
				disabled={isSubmitting}
				class="w-full"
				variant="secondary"
			>
				{#if isSubmitting}
					<Loader2 class="mr-2 h-4 w-4 animate-spin" />
					Submitting...
				{:else}
					<Send class="mr-2 h-4 w-4" />
					Submit Batch ({transactions.length} transaction{transactions.length !== 1 ? 's' : ''})
				{/if}
			</Button>
		</CardContent>
	</Card>

	<!-- Check Job Status -->
	<Card>
		<CardHeader>
			<CardTitle>Check Job Status</CardTitle>
			<CardDescription>Enter a job ID to check its status and results.</CardDescription>
		</CardHeader>
		<CardContent class="space-y-4">
			<div class="flex gap-2">
				<div class="flex-1">
					<Label for="job-id">Job ID</Label>
					<Input
						id="job-id"
						bind:value={jobId}
						placeholder="job_1732896000000_abc123"
						class="mt-2 font-mono"
					/>
				</div>
			</div>

			<Button
				onclick={checkJobStatus}
				disabled={isCheckingStatus || !jobId.trim()}
				class="w-full"
				variant="secondary"
			>
				{#if isCheckingStatus}
					<Loader2 class="mr-2 h-4 w-4 animate-spin" />
					Checking...
				{:else}
					<Search class="mr-2 h-4 w-4" />
					Check Status
				{/if}
			</Button>

			{#if jobStatus}
				<div class="mt-4 space-y-3 rounded-lg border p-4">
					{#if jobStatus.error}
						<div class="text-destructive text-sm font-medium">
							Error: {jobStatus.error}
						</div>
					{:else}
						<div>
							<span class="font-semibold">Job ID:</span>
							<span class="ml-2 font-mono text-sm">{jobStatus.id}</span>
						</div>

						<div>
							<span class="font-semibold">Status:</span>
							<span
								class={`ml-2 font-medium ${getStatusColor(jobStatus.status?.toString() || '')}`}
							>
								{jobStatus.status}
							</span>
						</div>

						{#if jobStatus.wallet}
							<div>
								<span class="font-semibold">Wallet:</span>
								<span class="ml-2 font-mono text-sm">{jobStatus.wallet}</span>
							</div>
						{/if}

						{#if jobStatus.balances}
							<div>
								<span class="font-semibold">Balances:</span>
								<div class="ml-2 text-sm">
									{#if jobStatus.balances.eth}
										<div>ETH: {jobStatus.balances.eth}</div>
									{/if}
									{#if jobStatus.balances.token}
										<div>Token: {jobStatus.balances.token}</div>
									{/if}
								</div>
							</div>
						{/if}

						{#if jobStatus.planned}
							<div>
								<span class="font-semibold">Planned:</span>
								<div class="ml-2 text-sm">
									<div>Count: {jobStatus.planned.count}</div>
									{#if jobStatus.planned.totalAmount}
										<div>Total Amount: {jobStatus.planned.totalAmount}</div>
									{/if}
								</div>
							</div>
						{/if}

						{#if jobStatus.completed !== undefined}
							<div>
								<span class="font-semibold">Completed:</span>
								<span class="ml-2 text-green-600">{jobStatus.completed}</span>
							</div>
						{/if}

						{#if jobStatus.transactions && jobStatus.transactions.length > 0}
							<div class="mt-4">
								<h4 class="mb-2 font-semibold">Transactions:</h4>
								<div class="max-h-64 space-y-2 overflow-y-auto">
									{#each jobStatus.transactions as tx, i (tx.hash || `status-tx-${i}`)}
										<div class="rounded border p-3 text-sm">
											<div class="mb-1 font-mono text-xs text-gray-500">Transaction #{i + 1}</div>
											{#if tx.hash}
												<div class="text-green-600">✓ Success</div>
												<div class="mt-1 break-all font-mono text-xs">
													Hash: {tx.hash}
												</div>
												{#if tx.status !== undefined}
													<div class="mt-1 text-xs">Status: {tx.status}</div>
												{/if}
											{:else if tx.error}
												<div class="text-red-600">✗ Failed</div>
												<div class="mt-1 text-xs text-red-500">{tx.error}</div>
											{/if}
										</div>
									{/each}
								</div>
							</div>
						{/if}

						{#if jobStatus.createdAt}
							<div class="text-xs text-gray-500">
								Created: {new Date(jobStatus.createdAt).toLocaleString()}
							</div>
						{/if}
					{/if}
				</div>
			{/if}
		</CardContent>
	</Card>
</div>

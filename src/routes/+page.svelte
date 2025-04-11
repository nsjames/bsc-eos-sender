<script lang="ts">
    import Papa from 'papaparse';
    import { ethers } from "ethers";
    import {parse} from "cookie";

    let csvFile = $state([]);
    let privateKey = $state('');
    let price = $state(0.99);

    let working = $state(false);
    let parsedData = $state(null);
    const tabledData = $derived.by(() => {
        if(!parsedData) return null;
        const totalEos = parsedData.reduce((acc, { eos }) => {
            return parseFloat(acc) + parseFloat(eos);
        }, 0);
        const totalUSD = parseFloat(totalEos) * price;


        const table = parsedData.map((row) => {
            return `<tr>
                <td class="px-2">${row.address}</td>
                <td class="px-2">${parseFloat(ethers.formatUnits(row.amount, 18)).toFixed(4)}</td>
                <td class="px-2">${parseFloat(ethers.formatUnits(row.amount, 18) * price).toFixed(2)}</td>
            </tr>`;
        }).join('');

        return `
<section class="p-4 mt-4 bg-gray-100 rounded">
    <h2 class="text-lg font-bold">Totals</h2>
    <p class="text-gray-700">Total EOS: ${totalEos}</p>
    <p class="text-gray-700">Total USD: $${totalUSD}</p>
</section>
<table class="w-full border-collapse border border-gray-200">
            <thead>
                <tr>
                    <th class="border border-gray-200 p-2">Address</th>
                    <th class="border border-gray-200 p-2">Amount</th>
                    <th class="border border-gray-200 p-2">USD</th>
                </tr>
            </thead>
            <tbody>
            <!-- total -->

            ${table}
</tbody>
        </table>`;
    });

    const process = async () => {


        try {

            const getCsvData = async () => {
                return new Promise((resolve, reject) => {
                    Papa.parse(csvFile[0], {
                        header: true,
                        skipEmptyLines: true,
                        complete: (results) => {
                            resolve(results.data);
                        },
                        error: (e) => {
                            reject(e);
                        }
                    });
                });
            };


            const csv = await getCsvData();
            const keys = Object.keys(csv[0]);
            parsedData = csv.map((row) => {
                const address = row[keys[0]];
                let amount = row[keys[1]];
                if (!ethers.isAddress(address)) {
                    console.error(`Invalid address: ${address}`);
                    return null;
                }
                let parsedAmount;
                let eosAmount;
                try {
                    amount = amount.replace('$', '');
                    amount = amount.replace(/,/g, '');
                    amount = amount.replace(/"/g, '');
                    eosAmount = parseFloat(amount);
                    parsedAmount = amount / price;
                    parsedAmount = ethers.parseUnits(parsedAmount.toString(), 18);
                    eosAmount = (parsedAmount / BigInt(10n ** 18n)).toString();
                } catch (e) {
                    console.error(`Invalid amount: ${amount}`, e);
                    return null;
                }
                return { address, amount: parsedAmount, eos:amount };
            }).filter(Boolean);


        } catch (e) {
            console.error(e);
            alert('Check the console for errors');
            // error = e.message;
            return;
        }

    }

    const reset = async () => {
        parsedData = null;
        csvFile = [];
        document.getElementById('input').value = '';
    }

    let responses:string[] = $state([]);
    let errors:boolean = $state(false);

    const send = async () => {
        if(working) return;
        working = true;

        const BSC_RPC = 'https://bsc-dataseed.bnbchain.org';
        const EOS_TOKEN_ADDRESS = '0x56b6fb708fc5732dec1afc8d8556423a2edccbd6';
        const ERC20_ABI = [
            'function transfer(address to, uint256 amount) returns (bool)',
            'function decimals() view returns (uint8)'
        ];

        const provider = new ethers.JsonRpcProvider(BSC_RPC);
        const wallet = new ethers.Wallet(privateKey, provider);
        const eosContract = new ethers.Contract(EOS_TOKEN_ADDRESS, ERC20_ABI, wallet);

        for (const { address, amount } of parsedData) {
            try {
                console.log(`Sending ${amount} EOS to ${address}`);
                const tx = await eosContract.transfer(address, amount);
                console.log(`✅ Sent | Tx: ${tx.hash}`);
                await tx.wait();
                responses.push(`✅ Sent ${amount} EOS to ${address} | Tx: ${tx.hash}`);
            } catch (err) {
                errors = true;
                console.error(`❌ Failed to send to ${address}: ${err.message}`);
                // responses.push(`❌ Failed to send ${amount} EOS to ${address}: ${err.message}`);
            }
        }
        working = false;
    }

    const canStart = $derived(csvFile.length && privateKey.length === 64 && price > 0);
</script>

<section class="flex flex-col max-w-2xl mx-auto p-4 mt-20">
    <label for="csv-file" class="text-lg font-bold">Upload CSV File</label>
    <input id="input" bind:files={csvFile} type="file" accept=".csv" />

    <label for="private-key" class="text-lg font-bold mt-4">Enter Private Key</label>
    <input bind:value={privateKey} type="password" placeholder="Enter private key" />

    <label for="price" class="text-lg font-bold mt-4">EOS/USD price</label>
    <input bind:value={price} type="number" placeholder="1.99" />

    {#if !parsedData}
        <button disabled={!canStart}
            class="mt-4 text-white py-2 px-4 rounded {canStart ? 'bg-blue-500 cursor-pointer' : 'bg-gray-500 opacity-20 cursor-not-allowed'}"
            onclick={process}>Start</button>
    {:else}
        <section class="flex gap-2">
            <button class="flex-1 mt-4 text-white py-2 px-4 rounded bg-red-500 cursor-pointer"
                    onclick={reset}>
                Reset
            </button>
            <button class="flex-1 mt-4 text-white py-2 px-4 rounded bg-blue-500 cursor-pointer"
                    onclick={send}>
                Start Sending
            </button>
        </section>
    {/if}

    {#if errors}
        <div class="mt-4 p-4 bg-red-500 text-white">
            You've got errors. Open the console to see.
        </div>
    {/if}

    {#if responses}
        <div class="mt-4 text-red-500">
            {#each responses as response}
                <p>{response}</p>
            {/each}
        </div>
    {/if}

    {#if tabledData}
        <section class="p-4 mt-4 bg-gray-100 rounded">
            {@html tabledData}
        </section>
    {/if}
</section>

<style>
    input {
        border: 1px solid #ccc;
        border-radius: 4px;
        padding: 8px;
        width: 100%;
    }
    input::file-selector-button {
        font-weight: bold;
        padding: 0.5em;
        border: thin solid grey;
        background-color: #e5e5e5;
        border-radius: 3px;
        margin-right:20px;
    }


    button {
        cursor: pointer;
    }

    /* add line to every table row */
    table tr {
        border-bottom: 1px solid rgba(0,0,0,0.1);
    }
</style>
<script setup>
import { ref, onMounted } from 'vue';
import { ethers } from 'ethers';
import { Line } from 'vue-chartjs';
import {
  CategoryScale,
  Chart as ChartJS,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  Title,
  Tooltip,
} from 'chart.js';
ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend
);

const provider = new ethers.providers.JsonRpcProvider(
  'https://testnet-rpc.callisto.network/'
);
const BLOCK_REWARD = 77.76;

const getTxDetails = (txHash) => {
  return new Promise(async (resolve, reject) => {
    const txDetailPro = provider.getTransaction(txHash);
    const txReceiptPro = provider.getTransactionReceipt(txHash);
    let txResume;
    try {
      const txDetails = await Promise.all([txDetailPro, txReceiptPro]);
      if (txDetails[0].type === 2) {
        txResume = {
          type: txDetails[0].type,
          maxPriorityFeePerGas: txDetails[0].maxPriorityFeePerGas,
          maxFeePerGas: txDetails[0].maxFeePerGas,
          gasUsed: txDetails[1].gasUsed,
        };
      } else {
        txResume = {
          type: txDetails[0].type,
        };
      }

      resolve(txResume);
    } catch (error) {
      reject('Problem in promise 😥');
      throw error;
    }
  });
};

const getBlockDetails = async (blockNumber) => {
  let burnedInThisBlock = ethers.BigNumber.from(0);
  let priorityFee = ethers.BigNumber.from(0);
  const block = await provider.getBlock(blockNumber);
  const size = (await provider.send('eth_getBlockByHash', [block.hash, false]))
    .size;
  const transactions = block.transactions;
  const baseFeePerGas = block.baseFeePerGas;
  let txDetailsPromises = [];
  for (const transaction of transactions) {
    const txDetails = getTxDetails(transaction);
    if (txDetails.type === 2) {
      txDetailsPromises.push(txDetails);
    }
  }
  const allTransactionDetails = await Promise.all(txDetailsPromises);
  for (const txDetail of allTransactionDetails) {
    if (
      txDetail.maxFeePerGas.sub(baseFeePerGas).gt(txDetail.maxPriorityFeePerGas)
    ) {
      priorityFee = priorityFee.add(
        txDetail.maxPriorityFeePerGas.mul(txDetail.gasUsed)
      );
    } else {
      priorityFee = priorityFee.add(
        txDetail.maxFeePerGas.sub(baseFeePerGas).mul(txDetail.gasUsed)
      );
    }

    burnedInThisBlock = burnedInThisBlock.add(
      baseFeePerGas.mul(txDetail.gasUsed)
    );
  }

  blockLabels.push(blockNumber);
  blockDataEmissions.push(
    BLOCK_REWARD - ethers.utils.formatEther(burnedInThisBlock)
  );
  blockDataPriorityFee.push(ethers.utils.formatEther(priorityFee));
  blockDataSize.push(ethers.utils.formatUnits(size, 0));

  // functions that mutate state and trigger updates
  dataEmissions.value = {
    labels: blockLabels,
    datasets: [
      {
        label: 'Emissions',
        backgroundColor: '#f87979',
        data: blockDataEmissions,
        borderColor: 'rgb(75, 192, 192)',
      },
    ],
  };

  dataPriorityFee.value = {
    labels: blockLabels,
    datasets: [
      {
        label: 'Priority Fee',
        backgroundColor: '#f87979',
        data: blockDataPriorityFee,
        borderColor: 'rgb(75, 192, 0)',
      },
    ],
  };

  dataSize.value = {
    labels: blockLabels,
    datasets: [
      {
        label: 'Size',
        backgroundColor: '#f87979',
        data: blockDataSize,
        borderColor: 'rgb(200, 0, 192)',
      },
    ],
  };
};

// reactive state
const dataEmissions = ref({
  datasets: [
    {
      data: [0],
    },
  ],
});

const dataPriorityFee = ref({
  datasets: [
    {
      data: [0],
    },
  ],
});

const dataSize = ref({
  datasets: [
    {
      data: [0],
    },
  ],
});

const options = {
  responsive: true,
  maintainAspectRatio: false,
  elements: {
    point: {
      radius: 0,
    },
  },
  plugins: {
    legend: {
      display: false,
    },
  },
  scales: {
    y: {
      grid: {
        color: 'grey',
      },
    },
    x: {
      grid: {
        color: 'grey',
      },
    },
  },
};

let blockLabels = [];
let blockDataEmissions = [];
let blockDataPriorityFee = [];
let blockDataSize = [];

// lifecycle hooks
onMounted(() => {
  provider.on('block', (blockNumber) => getBlockDetails(blockNumber));
});
</script>

<template>
  <div>
    <div>
      <v-row>
        <v-col>
          <v-card-title class="card-title"> Stats per Block </v-card-title>
        </v-col>
      </v-row>
    </div>
    <h1>Emissions</h1>
    <div><Line :data="dataEmissions" :options="options" /></div>
    <h1>Priority Fee</h1>
    <div><Line :data="dataPriorityFee" :options="options" /></div>
    <h1>Size (bytes)</h1>
    <div><Line :data="dataSize" :options="options" /></div>
  </div>
</template>
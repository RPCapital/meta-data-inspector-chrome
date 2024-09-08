<template>
    <main>
        <section>
            <h3>Meta Tags</h3>
            <p v-if="errorMessage" class="pico-background-pink-500 banner">{{ errorMessage }}</p>
            <p v-if="successMessage" class="pico-background-green-500 banner">{{ successMessage }}</p>
            <table>
                <thead>
                <tr>
                    <th>Name</th>
                    <th>Value</th>
                    <th>Actions</th>
                </tr>
                </thead>
                <tbody>
                <tr v-for="(item, index) in meta" :key="index">
                    <td>{{ item.name }}</td>
                    <td>{{ item.value }}</td>
                    <td>
                        <button style="padding-block: 4px;" :aria-busy="copyingIndex === index"
                                aria-label="Copy"
                                @click="copyToClipboard(item.html,
                        index)">
                            <span v-if="copyingIndex !== index">Copy</span>
                        </button>
                    </td>
                </tr>
                </tbody>
            </table>
        </section>
    </main>
</template>

<script setup>
import {onMounted, ref, watch} from 'vue';
const meta = ref([]);

chrome.runtime.onMessage.addListener(function (message) {
    console.log('message', message);
    if (message.action === 'fetched-data') {
        meta.value = message.data.meta;
    }
    return true;
});

onMounted(() => {
    fetchData();
});

const copyingIndex = ref(null);
const errorMessage = ref(null);
const successMessage = ref(null);

function copyToClipboard(html, index) {
    copyingIndex.value = index;
    navigator.clipboard.writeText(html).then(function () {
        setTimeout(() => {
            copyingIndex.value = null;
            successMessage.value = 'Copied to clipboard';
        }, 500);
    }, function (err) {
        errorMessage.value = err.message;
        setTimeout(() => {
            copyingIndex.value = null;
        }, 500);
    });
}

watch(successMessage, (value) => {
    if (value) {
        setTimeout(() => {
            successMessage.value = null;
        }, 3000);
    }
});

watch(errorMessage, (value) => {
    if (value) {
        setTimeout(() => {
            errorMessage.value = null;
        }, 3000);
    }
});

async function fetchData() {
    const [tab] = await chrome.tabs.query({active: true, currentWindow: true});
    chrome.scripting.executeScript({
        target: {tabId: tab.id},
        func: () => {
            const metaData = [];
            // fetch all tags from the head
            const head = document.head;
            const tags = head.querySelectorAll('meta');
            tags.forEach(tag => {
                const name = tag.getAttribute('name');
                const property = tag.getAttribute('property');
                const content = tag.getAttribute('content');
                const htmlAsString = tag.outerHTML;
                if (name || property) {
                    metaData.push({
                        name: name || property,
                        value: content,
                        html: htmlAsString,
                    });
                }
            });
            chrome.runtime.sendMessage({
                action: 'fetched-data',
                data: {
                    meta: metaData,
                }
            }, function (response) {
                // console.dir(response);
            });
        }
    });
}


</script>

<style scoped>
main {
    padding: 24px;
}

.banner {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    padding: 8px;
    text-align: center;
    font-weight: bold;
}
</style>

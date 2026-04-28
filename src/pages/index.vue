<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  stock: number;
}

interface CartItem {
  id: string;
  name: string;
  unitPrice: number;
  quantity: number;
}

interface Batch {
  id: number;
  items: CartItem[];
  total: number;
  time: string;
}

interface LogEntry {
  id: number;
  time: string;
  message: string;
}

// --- State ---
const inventory = ref<Product[]>([]);
const cart = ref<CartItem[]>([]);
const batches = ref<Batch[]>([]);
const activeCategory = ref('All');
const searchTerm = ref('');
const isStreamOpen = ref(false);
const activityLog = ref<LogEntry[]>([]);
const showPaymentModal = ref(false);
const showBatchModal = ref(false);
const showCartModal = ref(false);
const showReviewModal = ref(false);
const manualTotal = ref(0);

// --- Initialization ---
onMounted(async () => {
  try {
    const res = await fetch('/inventory.json');
    inventory.value = await res.json();
    logActivity("Inventory loaded. " + inventory.value.length + " items available.");
  } catch (e: any) {
    logActivity("Error loading inventory: " + e.message);
  }
});

// --- Activity Logging ---
function logActivity(message: string) {
  const time = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
  activityLog.value.push({
    id: Date.now(),
    time,
    message
  });
  // Auto-scroll logic could be added here in template
}

// --- Computed ---
const filteredInventory = computed(() => {
  return inventory.value.filter(item => {
    const matchesCat = activeCategory.value === 'All' || item.category === activeCategory.value;
    const matchesSearch = item.name.toLowerCase().includes(searchTerm.value.toLowerCase());
    return matchesCat && matchesSearch;
  });
});

const calculatedSubtotal = computed(() => {
  return cart.value.reduce((sum, item) => sum + (item.unitPrice * item.quantity), 0);
});

const cartTotal = computed(() => {
  // If we are in review or payment phase, use the manualTotal if set
  if (showReviewModal.value || showPaymentModal.value) {
    return manualTotal.value;
  }
  return calculatedSubtotal.value;
});

const cartItemCount = (id: string) => {
  return cart.value.filter(item => item.id === id).reduce((sum, item) => sum + item.quantity, 0);
};

// --- Methods ---
function addToCart(product: Product) {
  const existing = cart.value.find(item => item.id === product.id);
  if (existing) {
    existing.quantity++;
  } else {
    cart.value.push({
      id: product.id,
      name: product.name,
      unitPrice: product.price,
      quantity: 1
    });
  }
  logActivity(`Added ${product.name} to cart.`);
}

function updateQuantity(item: CartItem, delta: number) {
  item.quantity += delta;
  if (item.quantity <= 0) {
    removeFromCart(item.id);
  }
}

function removeFromCart(id: string) {
  const item = cart.value.find(i => i.id === id);
  if (item) {
    cart.value = cart.value.filter(i => i.id !== id);
    logActivity(`Removed ${item.name} from cart.`);
  }
}

function getEmoji(cat: string) {
  if (cat === 'Beverage') return '☕';
  if (cat === 'Food') return '🥐';
  return '📦';
}

function handlePaymentSuccess(method: string) {
  const total = cartTotal.value;
  logActivity(`Payment Success via ${method}: $${total.toFixed(2)} received.`);
  cart.value = [];
  showPaymentModal.value = false;
  manualTotal.value = 0;
  alert("Payment Successful!");
}

function saveBatch() {
  if (cart.value.length === 0) return;
  const total = calculatedSubtotal.value;
  batches.value.push({
    id: Date.now(),
    items: JSON.parse(JSON.stringify(cart.value)),
    total: total,
    time: new Date().toLocaleTimeString()
  });
  cart.value = [];
  logActivity(`Order saved to batch. Total: $${total.toFixed(2)}`);
  alert("Batch Saved!");
}

function recallBatch(batch: Batch) {
  cart.value = JSON.parse(JSON.stringify(batch.items));
  batches.value = batches.value.filter(b => b.id !== batch.id);
  showBatchModal.value = false;
  logActivity(`Batch #${batch.id.toString().slice(-4)} recalled.`);
}

function requestMoney() {
  if (cart.value.length === 0) {
    alert("Cart is empty");
    return;
  }
  manualTotal.value = calculatedSubtotal.value;
  showReviewModal.value = true;
}

function finalizeReview() {
  const adjustment = manualTotal.value - calculatedSubtotal.value;
  if (Math.abs(adjustment) > 0.001) {
    logActivity(`Total Adjusted by Cashier: $${calculatedSubtotal.value.toFixed(2)} -> $${manualTotal.value.toFixed(2)} (Diff: $${adjustment.toFixed(2)})`);
  }
  showReviewModal.value = false;
  showPaymentModal.value = true;
  logActivity("Money Request finalized for $" + manualTotal.value.toFixed(2));
}
</script>

<template>
  <div class="flex flex-col h-screen bg-[#0a0a0a] text-white font-sans overflow-hidden">
    <!-- Header / Search -->
    <header class="p-3 bg-[#161616] border-b border-[#262626] flex gap-2.5 items-center">
      <input
        v-model="searchTerm"
        type="text"
        class="bg-[#0a0a0a] border border-[#262626] rounded-lg p-2.5 text-sm flex-1 outline-none focus:border-[#00ff88]"
        placeholder="Search SKU or Product..."
      />
      <button 
        @click="isStreamOpen = !isStreamOpen"
        class="bg-[#161616] border border-[#262626] text-white py-2 px-3 rounded-lg cursor-pointer hover:bg-[#262626]"
      >
        Live
      </button>
    </header>

    <!-- Categories -->
    <div class="flex gap-2 p-3 overflow-x-auto whitespace-nowrap no-scrollbar">
      <div 
        v-for="cat in ['All', 'Beverage', 'Food']" 
        :key="cat"
        @click="activeCategory = cat"
        class="py-1.5 px-4 rounded-full text-xs border cursor-pointer transition-colors"
        :class="activeCategory === cat ? 'bg-[#00ff88] text-black border-[#00ff88] font-bold' : 'bg-[#161616] border-[#262626] text-[#888888]'"
      >
        {{ cat === 'All' ? 'All Items' : cat + 's' }}
      </div>
    </div>

    <!-- Product Grid -->
    <div class="flex-1 grid grid-cols-[repeat(auto-fill,minmax(100px,1fr))] gap-3 p-2 px-4 overflow-y-auto content-start">
      <div 
        v-for="item in filteredInventory" 
        :key="item.id"
        @click="addToCart(item)"
        class="bg-[#161616] rounded-lg border border-[#262626] p-2.5 text-center relative cursor-pointer active:scale-95 transition-transform"
      >
        <div v-if="cartItemCount(item.id) > 0" class="absolute -top-1 -right-1 bg-[#00ff88] text-black text-[0.6rem] py-0.5 px-1.5 rounded-full font-bold">
          {{ cartItemCount(item.id) }}
        </div>
        <div class="w-full aspect-square bg-gradient-to-br from-[#222] to-[#111] rounded-md mb-2 flex items-center justify-center text-2xl">
          {{ getEmoji(item.category) }}
        </div>
        <div class="text-[0.75rem] font-medium mb-1 line-clamp-2">
          {{ item.name }}
        </div>
        <div class="text-[0.8rem] text-[#00ff88] font-bold">
          ${{ item.price.toFixed(2) }}
        </div>
      </div>
    </div>

    <!-- Live Stream Panel -->
    <div 
      class="fixed top-0 bottom-0 w-[280px] bg-[#161616] border-l border-[#262626] transition-[right] duration-300 z-50 flex flex-col shadow-[-5px_0_15px_rgba(0,0,0,0.5)]"
      :class="isStreamOpen ? 'right-0' : '-right-[300px]'"
    >
      <div class="p-4 border-b border-[#262626] font-bold flex justify-between">
        <span>Activity Stream</span>
        <button @click="isStreamOpen = false" class="bg-transparent border-none text-white cursor-pointer text-xl">&times;</button>
      </div>
      <div class="flex-1 overflow-y-auto p-2.5 text-[0.7rem] flex flex-col">
        <div 
          v-for="log in activityLog" 
          :key="log.id"
          class="p-2 border-b border-[#262626] animate-fadeIn"
        >
          <strong class="mr-1">[{{ log.time }}]</strong> {{ log.message }}
        </div>
      </div>
    </div>

    <!-- Price Review Modal -->
    <div v-if="showReviewModal" class="fixed inset-0 bg-black/80 z-[200] flex items-center justify-center p-5">
      <div class="bg-[#161616] border border-[#262626] rounded-xl w-full max-w-[450px] flex flex-col max-h-[90vh]">
        <div class="p-5 border-b border-[#262626] flex justify-between items-center">
          <h2 class="text-xl font-bold">Review Order</h2>
          <button @click="showReviewModal = false" class="text-2xl">&times;</button>
        </div>
        <div class="flex-1 overflow-y-auto p-4">
          <!-- Item List (Read Only) -->
          <div v-for="item in cart" :key="item.id" class="mb-2 p-3 bg-[#222]/50 rounded-lg border border-[#333] flex justify-between items-center">
            <div>
              <div class="font-bold text-sm text-white">{{ item.name }} <span class="text-[#888] font-normal">x{{ item.quantity }}</span></div>
              <div class="text-[0.65rem] text-[#888] tracking-wider uppercase">${{ item.unitPrice.toFixed(2) }} UNIT PRICE</div>
            </div>
            <div class="text-sm font-bold text-[#888]">${{ (item.unitPrice * item.quantity).toFixed(2) }}</div>
          </div>

          <!-- Total Calculation -->
          <div class="mt-6 pt-4 border-t border-[#333]">
            <div class="flex justify-between items-center text-sm mb-4 px-1">
              <span class="text-[#888]">Calculated Subtotal</span>
              <span class="font-bold text-white">${{ calculatedSubtotal.toFixed(2) }}</span>
            </div>

            <div class="bg-[#222] p-5 rounded-xl border border-[#00ff88]/20">
              <label class="block text-[0.6rem] text-[#00ff88] font-black uppercase tracking-[0.2em] mb-2 text-center">Final Total to Charge</label>
              <div class="relative">
                <span class="absolute left-4 top-1/2 -translate-y-1/2 text-2xl text-[#888] font-bold">$</span>
                <input 
                  v-model.number="manualTotal" 
                  type="number" 
                  step="0.01"
                  class="w-full bg-[#0a0a0a] border border-[#00ff88]/30 rounded-xl py-5 pl-10 pr-4 text-4xl font-black text-[#00ff88] text-center outline-none focus:border-[#00ff88] transition-all"
                  autofocus
                />
              </div>
              <div v-if="Math.abs(manualTotal - calculatedSubtotal) > 0.01" class="mt-3 text-center text-xs font-bold text-orange-400">
                Adjustment: {{ manualTotal > calculatedSubtotal ? '+' : '-' }} ${{ Math.abs(manualTotal - calculatedSubtotal).toFixed(2) }}
              </div>
            </div>
          </div>
        </div>
        <div class="p-5 border-t border-[#262626] bg-[#1a1a1a] rounded-b-xl flex gap-3">
          <button @click="showReviewModal = false" class="flex-1 bg-[#333] py-4 rounded-xl font-bold hover:bg-[#444] transition-colors text-sm">Cancel</button>
          <button @click="finalizeReview" class="flex-2 bg-[#00ff88] text-black py-4 rounded-xl font-black uppercase tracking-wider shadow-[0_4px_20px_rgba(0,255,136,0.15)] hover:bg-[#00e57a] transition-all active:scale-[0.98]">
            Finalize Charge
          </button>
        </div>
      </div>
    </div>

    <!-- Edit Cart Modal -->
    <div v-if="showCartModal" class="fixed inset-0 bg-black/80 z-[190] flex items-center justify-center p-5">
      <div class="bg-[#161616] border border-[#262626] rounded-xl w-full max-w-[400px] flex flex-col max-h-[80vh]">
        <div class="p-5 border-b border-[#262626] flex justify-between items-center">
          <h2 class="text-xl font-bold">Cart Contents</h2>
          <button @click="showCartModal = false" class="text-2xl">&times;</button>
        </div>
        <div class="flex-1 overflow-y-auto p-2">
          <div v-if="cart.length === 0" class="p-10 text-center text-[#888]">
            Your cart is empty.
          </div>
          <div 
            v-for="item in cart" 
            :key="item.id"
            class="p-3 mb-2 bg-[#222] rounded-lg flex justify-between items-center"
          >
            <div class="flex-1">
              <div class="font-bold text-sm">{{ item.name }}</div>
              <div class="text-[#00ff88] text-xs font-bold">${{ (item.price * item.quantity).toFixed(2) }}</div>
            </div>
            
            <div class="flex items-center gap-3">
              <div class="flex items-center bg-[#111] rounded-lg border border-[#333] p-1">
                <button @click="updateQuantity(item, -1)" class="w-8 h-8 flex items-center justify-center text-[#888] hover:text-white transition-colors">-</button>
                <span class="w-8 text-center font-bold text-sm">{{ item.quantity }}</span>
                <button @click="updateQuantity(item, 1)" class="w-8 h-8 flex items-center justify-center text-[#888] hover:text-white transition-colors">+</button>
              </div>
              
              <button @click="removeFromCart(item.id)" class="bg-red-500/10 text-red-500 p-2 rounded-lg hover:bg-red-500/20 transition-colors">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" viewBox="0 0 16 16">
                  <path d="M5.5 5.5A.5.5 0 0 1 6 6v6a.5.5 0 0 1-1 0V6a.5.5 0 0 1 .5-.5zm2.5 0a.5.5 0 0 1 .5.5v6a.5.5 0 0 1-1 0V6a.5.5 0 0 1 .5-.5zm3 .5a.5.5 0 0 0-1 0v6a.5.5 0 0 0 1 0V6z"/>
                  <path fill-rule="evenodd" d="M14.5 3a1 1 0 0 1-1 1H13v9a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V4h-.5a1 1 0 0 1-1-1V2a1 1 0 0 1 1-1H6a1 1 0 0 1 1-1h2a1 1 0 0 1 1 1h3.5a1 1 0 0 1 1 1v1zM4.118 4L4 4.059V13a1 1 0 0 0 1 1h6a1 1 0 0 0 1-1V4.059L11.882 4H4.118zM2.5 3V2h11v1h-11z"/>
                </svg>
              </button>
            </div>
          </div>
        </div>
        <div class="p-5 border-t border-[#262626] bg-[#1a1a1a] rounded-b-xl">
          <div class="flex justify-between items-center mb-4">
            <span class="text-[#888]">Subtotal</span>
            <span class="text-xl font-bold text-[#00ff88]">${{ cartTotal.toFixed(2) }}</span>
          </div>
          <button @click="showCartModal = false" class="w-full bg-[#333] py-3 rounded-lg font-bold">Close Editor</button>
        </div>
      </div>
    </div>

    <!-- Payment Modal -->
    <div v-if="showPaymentModal" class="fixed inset-0 bg-black/80 z-[200] flex items-center justify-center p-5">
      <div class="bg-[#161616] border border-[#262626] rounded-xl w-full max-w-[400px] p-6 text-center">
        <h2>Requesting Payment</h2>
        <div class="w-[200px] h-[200px] bg-white mx-auto my-5 p-2.5 rounded-lg flex flex-col items-center justify-center text-black">
          <div class="text-[3rem] mb-2.5">📱</div>
          <div class="text-sm">Scan QR to pay</div>
        </div>
        <p class="text-2xl font-bold mb-5">${{ cartTotal.toFixed(2) }}</p>
        <div class="flex gap-2.5 mb-2.5">
          <button @click="handlePaymentSuccess('NFC Tap')" class="flex-1 bg-[#333] text-white border-none py-3 px-4 rounded-lg font-semibold cursor-pointer hover:bg-[#444]">
            Simulate NFC Tap
          </button>
          <button @click="handlePaymentSuccess('QR Scan')" class="flex-1 bg-[#333] text-white border-none py-3 px-4 rounded-lg font-semibold cursor-pointer hover:bg-[#444]">
            Simulate QR Scan
          </button>
        </div>
        <button @click="showPaymentModal = false" class="w-full bg-[#333] text-white border-none py-3 px-4 rounded-lg font-semibold cursor-pointer hover:bg-[#444]">
          Cancel
        </button>
      </div>
    </div>

    <!-- Batch Modal -->
    <div v-if="showBatchModal" class="fixed inset-0 bg-black/80 z-[200] flex items-center justify-center p-5">
      <div class="bg-[#161616] border border-[#262626] rounded-xl w-full max-w-[400px] p-6 text-center">
        <h2 class="mb-4">Saved Batches</h2>
        <div class="text-left max-h-[300px] overflow-y-auto">
          <div v-if="batches.length === 0" class="text-center text-[#888]">No saved batches.</div>
          <div 
            v-for="batch in batches" 
            :key="batch.id"
            @click="recallBatch(batch)"
            class="p-3 border-b border-[#262626] flex justify-between items-center cursor-pointer hover:bg-[#222]"
          >
            <div>
              <strong class="block text-sm">Batch #{{ batch.id.toString().slice(-4) }}</strong>
              <small class="text-[#888]">{{ batch.items.length }} items - {{ batch.time }}</small>
            </div>
            <strong class="text-[#00ff88]">${{ batch.total.toFixed(2) }}</strong>
          </div>
        </div>
        <button @click="showBatchModal = false" class="mt-5 w-full bg-[#333] text-white border-none py-3 px-4 rounded-lg font-semibold cursor-pointer hover:bg-[#444]">
          Close
        </button>
      </div>
    </div>

    <!-- Footer Cart & Action -->
    <footer class="bg-[#161616] p-4 border-t border-[#262626] flex items-center gap-3">
      <div @click="showCartModal = true" class="flex-1 cursor-pointer hover:bg-[#262626] p-1 rounded-lg transition-colors">
        <span class="text-[0.7rem] text-[#888888] block">{{ cart.length }} Items in cart (Edit)</span>
        <span class="text-[1.1rem] font-extrabold block">${{ cartTotal.toFixed(2) }}</span>
      </div>
      <div class="flex gap-2">
        <button @click="saveBatch" class="bg-[#333] text-white border-none py-3 px-4 rounded-lg font-semibold cursor-pointer hover:bg-[#444] text-xs">
          Save Batch
        </button>
        <button @click="showBatchModal = true" class="bg-[#333] text-white border-none py-3 px-4 rounded-lg font-semibold cursor-pointer hover:bg-[#444] text-xs">
          Batches
        </button>
        <button 
          @click="requestMoney"
          class="bg-[#00ff88] text-black border-none py-3 px-5 rounded-lg font-black uppercase text-[0.85rem] shadow-[0_4px_15px_rgba(0,255,136,0.2)] cursor-pointer hover:bg-[#00e57a]"
        >
          Request Money
        </button>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.no-scrollbar::-webkit-scrollbar {
  display: none;
}
.no-scrollbar {
  -ms-overflow-style: none;
  scrollbar-width: none;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(5px); }
  to { opacity: 1; transform: translateY(0); }
}
.animate-fadeIn {
  animation: fadeIn 0.3s ease-out forwards;
}
</style>

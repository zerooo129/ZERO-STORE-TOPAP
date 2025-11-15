# ZERO-STORE-TOPAP/*
Zero Store - Top Up (Single-file React component)
Updated: added clickable payment methods, icons, QR simulation (DANA/GoPay), pulsa & bank transfer flows, and example backend structure for payment gateway (Midtrans) integration.

How to use:
- Frontend: this React component (requires Tailwind + lucide-react). It calls backend endpoints for real payments:
  - POST /api/create-payment  -> creates payment order (Midtrans/Xendit) or returns QR/VA data
  - POST /api/verify-payment  -> verify webhook or status

- Backend example (Node/Express) snippet is included below in comments. Replace with your gateway credentials and endpoints.

Notes:
- The QR simulation shows a generated data-URI (placeholder). Replace with real QR image from gateway when integrating.
- For Pulsa (carrier billing) and bank transfer, the backend typically creates a VA or transaction and returns account details.
- Security: keep server keys secret, use HTTPS, validate webhooks.
*/

import React, { useState } from 'react';
import { ShoppingCart, Search, CreditCard, X, Check, Hash, Phone, ImageIcon } from 'lucide-react';

export default function ZeroStore() {
  const [query, setQuery] = useState('');
  const [selectedGame, setSelectedGame] = useState(null);
  const [cart, setCart] = useState([]);
  const [showCheckout, setShowCheckout] = useState(false);
  const [buyer, setBuyer] = useState({ name: '', phone: '' });
  const [orderSuccess, setOrderSuccess] = useState(null);
  const [paymentMethod, setPaymentMethod] = useState('qr_dana');
  const [qrDataUri, setQrDataUri] = useState(null);
  const [processingPayment, setProcessingPayment] = useState(false);
  const [bankVirtualAccount, setBankVirtualAccount] = useState(null);

  // ... games and denominations same as before (omitted here for brevity) -- keep full lists as in previous file
  const games = [
    { id: 'roblox', name: 'Roblox', category: 'Casual', color: 'bg-purple-500' },
    { id: 'freefire', name: 'Free Fire', category: 'Battle Royale', color: 'bg-orange-400' },
    { id: 'ml', name: 'Mobile Legends', category: 'MOBA', color: 'bg-pink-500' },
    { id: 'pubg', name: 'PUBG', category: 'Battle Royale', color: 'bg-yellow-500' },
    { id: 'valorant', name: 'Valorant', category: 'FPS', color: 'bg-indigo-600' },
    { id: 'genshin', name: 'Genshin', category: 'RPG', color: 'bg-teal-500' },
  ];

  const denominations = {
    roblox: [
      { id: 'r1', title: '50 Robux', price: 12000 },
      { id: 'r2', title: '125 Robux', price: 30000 },
      { id: 'r3', title: '300 Robux', price: 70000 },
    ],
    freefire: [
      { id: 'ff1', title: 'Diamond 50', price: 8000 },
      { id: 'ff2', title: 'Diamond 140', price: 20000 },
      { id: 'ff3', title: 'Diamond 355', price: 48000 },
    ],
    ml: [
      { id: 'ml1', title: 'Diamond 12', price: 9000 },
      { id: 'ml2', title: 'Diamond 50', price: 35000 },
      { id: 'ml3', title: 'Diamond 113', price: 75000 },
    ],
    pubg: [
      { id: 'p1', title: 'UC 60', price: 15000 },
      { id: 'p2', title: 'UC 300', price: 70000 },
    ],
    valorant: [
      { id: 'v1', title: 'Radiante 475', price: 150000 },
      { id: 'v2', title: 'Radiante 1000', price: 300000 },
    ],
    genshin: [
      { id: 'g1', title: 'Genesis Crystal 60', price: 25000 },
      { id: 'g2', title: 'Genesis Crystal 300', price: 115000 },
    ],
  };

  function priceFormat(v) {
    return 'Rp ' + v.toLocaleString('id-ID');
  }

  function addToCart(gameId, denom) {
    const item = { gameId, denom, qty: 1, key: `${gameId}_${denom.id}` };
    setCart((c) => {
      const exists = c.find((x) => x.key === item.key);
      if (exists) return c.map((x) => (x.key === item.key ? { ...x, qty: x.qty + 1 } : x));
      return [...c, item];
    });
    setOrderSuccess(null);
  }

  function updateQty(key, delta) {
    setCart((c) =>
      c
        .map((it) => (it.key === key ? { ...it, qty: Math.max(0, it.qty + delta) } : it))
        .filter((it) => it.qty > 0)
    );
  }

  function removeFromCart(key) {
    setCart((c) => c.filter((it) => it.key !== key));
  }

  function cartTotal() {
    return cart.reduce((sum, it) => {
      const price = denominations[it.gameId].find((d) => d.id === it.denom.id).price;
      return sum + price * it.qty;
    }, 0);
  }

  async function processPayment() {
    if (!buyer.name || !buyer.phone) {
      alert('Masukkan nama dan nomor ponsel pembeli');
      return;
    }
    if (cart.length === 0) {
      alert('Keranjang kosong');
      return;
    }

    setProcessingPayment(true);
    setQrDataUri(null);
    setBankVirtualAccount(null);

    const payload = {
      buyer,
      items: cart,
      method: paymentMethod,
      total: cartTotal(),
    };

    try {
      // The frontend calls a backend endpoint to create payment.
      // Example: POST /api/create-payment -> returns { success:true, type:'qr', qr:'data:image/png;base64,...' } or { type:'va', va_account:'123456', bank:'BCA' }
      const res = await fetch('/api/create-payment', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
      const data = await res.json();

      if (data.success) {
        if (data.type === 'qr') {
          setQrDataUri(data.qr);
        } else if (data.type === 'va') {
          setBankVirtualAccount({ account: data.va_account, bank: data.bank, expires: data.expires });
        } else if (data.type === 'pulsa') {
          // pulsa provider response handling
          alert('Permintaan pulsa dikirim. Cek status pesanan di riwayat.');
        }

        // For demo, mark order success (real flow waits for payment confirmation)
        const order = { id: `ORD-${Date.now()}`, buyer, items: cart, total: cartTotal(), createdAt: new Date().toISOString() };
        setOrderSuccess(order);
        setCart([]);
      } else {
        alert('Gagal membuat pembayaran: ' + (data.message || 'unknown'));
      }
    } catch (err) {
      console.error(err);
      alert('Terjadi kesalahan saat memproses pembayaran');
    }

    setProcessingPayment(false);
  }

  const filteredGames = games.filter((g) => g.name.toLowerCase().includes(query.toLowerCase()));

  return (
    <div className="min-h-screen bg-slate-50 p-6">
      <header className="max-w-6xl mx-auto flex items-center justify-between">
        <div className="flex items-center gap-4">
          <div className="rounded-full bg-black text-white w-12 h-12 flex items-center justify-center font-bold">Z</div>
          <div>
            <h1 className="text-2xl font-extrabold">Zero Store</h1>
            <p className="text-sm text-slate-500">Top-up game cepat & aman</p>
          </div>
        </div>
        <div className="flex items-center gap-4">
          <div className="relative">
            <input
              type="text"
              placeholder="Cari game atau item..."
              className="pl-10 pr-4 py-2 rounded-lg border w-64 bg-white"
              value={query}
              onChange={(e) => setQuery(e.target.value)}
            />
            <div className="absolute left-3 top-2.5 opacity-60">
              <Search size={18} />
            </div>
          </div>

          <button
            onClick={() => setShowCheckout(true)}
            className="flex items-center gap-2 bg-white border px-3 py-2 rounded-lg shadow-sm"
          >
            <ShoppingCart size={16} />
            <span className="text-sm font-medium">Cart ({cart.length})</span>
          </button>
        </div>
      </header>

      <main className="max-w-6xl mx-auto mt-8 grid grid-cols-1 md:grid-cols-4 gap-6">
        <aside className="md:col-span-1 bg-white rounded-xl p-4 shadow-sm">
          <h2 className="font-semibold mb-3">Kategori Game</h2>
          <div className="flex flex-col gap-2">
            {games.map((g) => (
              <button
                key={g.id}
                onClick={() => setSelectedGame(g.id === selectedGame ? null : g.id)}
                className={`flex items-center justify-between p-2 rounded-md hover:bg-slate-50 ${selectedGame === g.id ? 'ring-2 ring-indigo-300' : ''}`}
              >
                <div className="flex items-center gap-3">
                  <div className={`${g.color} w-8 h-8 rounded flex items-center justify-center text-white font-bold`}>{g.name[0]}</div>
                  <div>
                    <div className="font-medium">{g.name}</div>
                    <div className="text-xs text-slate-400">{g.category}</div>
                  </div>
                </div>
                <div className="text-xs text-slate-400">Pilih</div>
              </button>
            ))}
          </div>
        </aside>

        <section className="md:col-span-3">
          <div className="bg-white rounded-xl p-4 shadow-sm">
            <div className="flex items-center justify-between">
              <h2 className="text-xl font-semibold">Top Up Populer</h2>
              <div className="text-sm text-slate-500">Zero Store • cepat aman • layanan 24/7</div>
            </div>

            <div className="mt-4 grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
              {filteredGames.map((g) => (
                <div key={g.id} className="border rounded-lg p-4 flex flex-col">
                  <div className="flex items-center gap-4">
                    <div className={`${g.color} w-14 h-14 rounded-lg flex items-center justify-center text-white font-bold text-lg`}>{g.name[0]}</div>
                    <div>
                      <div className="font-semibold">{g.name}</div>
                      <div className="text-xs text-slate-400">{g.category}</div>
                    </div>
                  </div>

                  <div className="mt-4">
                    <div className="text-sm font-medium mb-2">Pilih Denominasi</div>
                    <div className="flex flex-col gap-2">
                      {(denominations[g.id] || []).map((d) => (
                        <div key={d.id} className="flex items-center justify-between border rounded p-2">
                          <div>
                            <div className="text-sm font-medium">{d.title}</div>
                            <div className="text-xs text-slate-400">{priceFormat(d.price)}</div>
                          </div>
                          <button onClick={() => addToCart(g.id, d)} className="px-3 py-1 rounded bg-indigo-600 text-white text-sm">Beli</button>
                        </div>
                      ))}
                      {(!denominations[g.id] || denominations[g.id].length === 0) && <div className="text-xs text-slate-400">Tidak ada paket saat ini.</div>}
                    </div>
                  </div>
                </div>
              ))}
            </div>
          </div>

          <div className="mt-6 bg-white rounded-xl p-4 shadow-sm">
            <h3 className="font-semibold">Cara Top Up</h3>
            <ol className="mt-2 list-decimal ml-5 text-sm text-slate-600">
              <li>Pilih game dan paket</li>
              <li>Masukkan ID atau username pemain saat checkout</li>
              <li>Pilih metode pembayaran dan lakukan pembayaran</li>
              <li>Pesanan diproses otomatis (atau manual dalam beberapa menit)</li>
            </ol>
          </div>
        </section>
      </main>

      {/* Checkout Drawer / Modal */}
      {showCheckout && (
        <div className="fixed inset-0 bg-black/40 flex items-end md:items-center justify-center z-50">
          <div className="bg-white rounded-t-xl md:rounded-xl w-full md:w-3/5 p-6 shadow-xl">
            <div className="flex items-center justify-between">
              <h3 className="text-lg font-semibold">Checkout</h3>
              <button onClick={() => setShowCheckout(false)} className="p-2 rounded-md hover:bg-slate-100">
                <X size={18} />
              </button>
            </div>

            <div className="mt-4 grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label className="block text-sm font-medium">Nama</label>
                <input value={buyer.name} onChange={(e) => setBuyer({ ...buyer, name: e.target.value })} className="mt-1 w-full border rounded px-3 py-2" placeholder="Nama pembeli" />

                <label className="block text-sm font-medium mt-3">No. HP / ID Player</label>
                <input value={buyer.phone} onChange={(e) => setBuyer({ ...buyer, phone: e.target.value })} className="mt-1 w-full border rounded px-3 py-2" placeholder="08xxxxxxxx" />

                <div className="mt-3 text-sm text-slate-500">Pilih Metode Pembayaran</div>

                <div className="mt-2 grid grid-cols-2 gap-2">
                  <button onClick={() => setPaymentMethod('qr_dana')} className={`p-2 rounded border ${paymentMethod==='qr_dana'?'ring-2 ring-indigo-300':''}`}>
                    <div className="flex items-center gap-2"><ImageIcon size={16}/> QR DANA</div>
                  </button>
                  <button onClick={() => setPaymentMethod('qr_gopay')} className={`p-2 rounded border ${paymentMethod==='qr_gopay'?'ring-2 ring-indigo-300':''}`}>
                    <div className="flex items-center gap-2"><ImageIcon size={16}/> GoPay</div>
                  </button>
                  <button onClick={() => setPaymentMethod('pulsa')} className={`p-2 rounded border ${paymentMethod==='pulsa'?'ring-2 ring-indigo-300':''}`}>
                    <div className="flex items-center gap-2"><Phone size={16}/> Pulsa</div>
                  </button>
                  <button onClick={() => setPaymentMethod('bank_va')} className={`p-2 rounded border ${paymentMethod==='bank_va'?'ring-2 ring-indigo-300':''}`}>
                    <div className="flex items-center gap-2"><Hash size={16}/> Transfer Bank</div>
                  </button>
                </div>

                <div className="mt-3 text-xs text-slate-500">Catatan: QR -> scan dengan aplikasi DANA / GoPay. Pulsa -> pemotongan pulsa otomatis provider (jika tersedia). Bank -> virtual account akan muncul setelah klik Bayar.</div>

              </div>

              <div>
                <h4 className="font-medium">Ringkasan Pesanan</h4>
                <div className="mt-2 space-y-2 max-h-48 overflow-auto">
                  {cart.length === 0 && <div className="text-sm text-slate-400">Keranjang kosong.</div>}
                  {cart.map((it) => {
                    const g = games.find((x) => x.id === it.gameId);
                    const d = denominations[it.gameId].find((x) => x.id === it.denom.id);
                    return (
                      <div key={it.key} className="flex items-center justify-between border rounded p-2">
                        <div>
                          <div className="text-sm font-medium">{g.name} — {d.title}</div>
                          <div className="text-xs text-slate-400">{priceFormat(d.price)} x {it.qty}</div>
                        </div>
                        <div className="flex items-center gap-2">
                          <button onClick={() => updateQty(it.key, -1)} className="px-2 py-1 border rounded">-</button>
                          <div className="w-6 text-center">{it.qty}</div>
                          <button onClick={() => updateQty(it.key, +1)} className="px-2 py-1 border rounded">+</button>
                          <button onClick={() => removeFromCart(it.key)} className="ml-2 text-sm text-red-500">hapus</button>
                        </div>
                      </div>
                    );
                  })}
                </div>

                <div className="mt-4 flex items-center justify-between">
                  <div className="text-sm text-slate-500">Total</div>
                  <div className="font-semibold">{priceFormat(cartTotal())}</div>
                </div>

                <div className="mt-4 flex gap-2">
                  <button onClick={processPayment} disabled={processingPayment} className="flex-1 px-4 py-2 rounded bg-emerald-600 text-white flex items-center justify-center gap-2">
                    <CreditCard size={16} /> <span>{processingPayment ? 'Memproses...' : 'Bayar Sekarang'}</span>
                  </button>
                  <button onClick={() => setShowCheckout(false)} className="px-4 py-2 rounded border">Batal</button>
                </div>

                {/* Show QR or VA if available */}
                {qrDataUri && (
                  <div className="mt-4 border rounded p-3 text-center">
                    <div className="font-medium">Scan QR untuk membayar</div>
                    <img src={qrDataUri} alt="QR Payment" className="mx-auto mt-2 w-40 h-40" />
                    <div className="text-xs text-slate-500 mt-2">QR berlaku 15 menit. Setelah bayar, kami akan memproses pesanan.</div>
                  </div>
                )}

                {bankVirtualAccount && (
                  <div className="mt-4 border rounded p-3">
                    <div className="font-medium">Virtual Account</div>
                    <div className="mt-1 text-sm">Bank: {bankVirtualAccount.bank}</div>
                    <div className="text-sm">Nomor: {bankVirtualAccount.account}</div>
                    <div className="text-xs text-slate-500">Kadaluarsa: {bankVirtualAccount.expires}</div>
                  </div>
                )}

              </div>
            </div>
          </div>
        </div>
      )}

      {/* Order success toast/modal */}
      {orderSuccess && (
        <div className="fixed right-6 bottom-6 bg-white shadow-lg rounded-lg p-4 z-50 w-80">
          <div className="flex items-start gap-3">
            <div className="p-2 rounded bg-emerald-100">
              <Check size={18} className="text-emerald-600" />
            </div>
            <div>
              <div className="font-semibold">Pesanan Berhasil</div>
              <div className="text-sm text-slate-600">ID: {orderSuccess.id}</div>
              <div className="text-sm text-slate-600">Total: {priceFormat(orderSuccess.total)}</div>
              <div className="mt-2 text-xs text-slate-500">Terima kasih! Tim kami akan memproses pesanan. Simpan ID pesanan bila diperlukan.</div>
            </div>
            <div className="ml-auto">
              <button onClick={() => setOrderSuccess(null)} className="p-1">
                <X size={16} />
              </button>
            </div>
          </div>
        </div>
      )}

      <footer className="max-w-6xl mx-auto mt-10 text-center text-sm text-slate-500">
        © {new Date().getFullYear()} Zero Store — Top Up Games • Demo UI
      </footer>
    </div>
  );
}

/*
Backend example (Node/Express) - save as server.js (simplified)

const express = require('express');
const app = express();
app.use(express.json());

// NOTE: Replace with your Midtrans/Xendit SDK setup and credentials
app.post('/api/create-payment', async (req, res) => {
  const { buyer, items, method, total } = req.body;
  // create order in DB, then call gateway API

  if (method === 'qr_dana' || method === 'qr_gopay') {
    // Example: call gateway to create QR payload or generate QR content
    // For demo return a placeholder image (1x1 png base64)
    const placeholder = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR4nGNgYAAAAAMAAWgmWQ0AAAAASUVORK5CYII=';
    return res.json({ success:true, type:'qr', qr: placeholder });
  }

  if (method === 'bank_va') {
    // create VA via gateway -> return account number and bank
    return res.json({ success:true, type:'va', bank:'BCA', va_account:'1234567890', expires: '2025-11-30 23:59' });
  }

  if (method === 'pulsa') {
    // call pulsa provider API
    return res.json({ success:true, type:'pulsa', message:'Permintaan pulsa dikirim ke provider' });
  }

  res.json({ success:false, message:'Metode tidak didukung' });
});

app.listen(3000, ()=>console.log('Server running on 3000'));

Security & production:
- Store gateway server keys in env vars
- Validate and handle webhooks from gateway to confirm payments
- Use HTTPS and CORS properly
- Implement retry & idempotency for payment creation

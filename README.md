<!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Cardápio Digital – Vitória's Pizzaria</title>
  <style>
    /* — estilo minimalista, moderno e responsivo — */
    :root {
      --bg: #0b0b0b;
      --card: #111;
      --accent: #f58220;
      --text: #ddd;
      --white: #fff;
    }
    * { box-sizing: border-box; }
    body {
      margin: 0;
      font-family: sans-serif;
      background: var(--bg);
      color: var(--text);
      line-height: 1.4;
    }
    header {
      background: linear-gradient(90deg, #0f0f0f, #111);
      color: var(--white);
      padding: 16px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    header h1 { margin: 0; font-size: 20px; }
    header p { margin: 0; font-size: 14px; }
    .container {
      display: grid;
      grid-template-columns: 1fr 300px;
      gap: 16px;
      padding: 16px;
      max-width: 1000px;
      margin: auto;
    }
    @media (max-width: 800px) {
      .container { grid-template-columns: 1fr; }
    }
    .category h2 {
      border-bottom: 2px solid var(--accent);
      padding-bottom: 4px;
      color: var(--white);
    }
    .product {
      background: var(--card);
      padding: 10px;
      border-radius: 6px;
      margin-bottom: 10px;
    }
    .product h3 { margin: 0 0 6px; color: var(--white); }
    .prices { color: var(--accent); font-weight: bold; }
    .btn {
      background: var(--accent);
      color: var(--bg);
      border: none;
      padding: 8px 12px;
      border-radius: 5px;
      cursor: pointer;
      font-weight: bold;
    }
    aside.cart {
      background: var(--card);
      padding: 16px;
      border-radius: 8px;
    }
    .field input, .field textarea, .field select {
      width: 100%;
      padding: 8px;
      background: #222;
      border: none;
      border-radius: 4px;
      color: var(--text);
      margin-bottom: 8px;
    }
  </style>
</head>
<body>
  <header>
    <h1>Vitória's Pizzaria • Forno a Lenha</h1>
    <p>Compre uma pizza e ganhe 1 refrigerante grátis!</p>
  </header>

  <div class="container">
    <main id="menuArea">
      <p>Carregando cardápio...</p>
    </main>

    <aside class="cart">
      <h2>Carrinho</h2>
      <div id="cartItems">Nenhum item adicionado.</div>
      <p><strong>Total: </strong><span id="cartTotal">R$ 0,00</span></p>
      <div class="field"><input id="custName" placeholder="Nome" /></div>
      <div class="field"><input id="custPhone" placeholder="Telefone (WhatsApp)" /></div>
      <div class="field"><textarea id="custAddress" rows="2" placeholder="Endereço"></textarea></div>
      <div class="field">
        <select id="payMethod">
          <option>Dinheiro</option>
          <option>Cartão no Local</option>
          <option>Pix</option>
        </select>
      </div>
      <button class="btn" id="sendWhats">Finalizar pelo WhatsApp</button>
    </aside>
  </div>

<script>
  const SHEET_ID = '2PACX-1vT5ANvFSR9RaRPgciR8ztS1Jwz5SOk5jpMZUv4EHcoKwPR3DiymOzSaIr580sev7XWVVHkE6WiVu1Bn';
  const SHEET_NAME = 'Sheet1'; // Altere se sua aba tiver outro nome
  const WHATS_NUM = '+5512988874265';

  const menuArea = document.getElementById('menuArea');
  const cartItemsEl = document.getElementById('cartItems');
  const cartTotalEl = document.getElementById('cartTotal');
  let cart = [];

  function formatBRL(v) {
    return 'R$ ' + v.toFixed(2).replace('.', ',');
  }

  function fetchSheet() {
    const url = `https://docs.google.com/spreadsheets/d/${SHEET_ID}/gviz/tq?tqx=out:json&sheet=${encodeURIComponent(SHEET_NAME)}`;
    return fetch(url).then(r => r.text()).then(txt => {
      const json = JSON.parse(txt.match(/google\\.visualization\\.Query\\.setResponse\\((.*)\\);/)[1]);
      const rows = json.table.rows;
      const cols = json.table.cols.map(c => c.label || c.id);
      return rows.map(r => {
        let obj = {};
        r.c.forEach((cell, i) => obj[cols[i]] = cell ? cell.v : '');
        return obj;
      });
    });
  }

  function loadMenu() {
    fetchSheet().then(data => {
      menuArea.innerHTML = '';
      const groups = {};
      data.forEach(item => {
        const cat = item.Categoria || item.Category;
        if (!groups[cat]) groups[cat] = [];
        groups[cat].push(item);
      });
      for (const cat in groups) {
        const div = document.createElement('div');
        div.innerHTML = `<h2>${cat}</h2>`;
        groups[cat].forEach(it => {
          const p = document.createElement('div');
          const price8 = it['8Fatias'] || it['Preço8'] || it['Price8'] || '';
          const price4 = it['4Fatias'] || it['Preço4'] || it['Price4'] || '';
          p.className = 'product';
          p.innerHTML = `<h3>${it.Nome || it.Name}</h3>
                         <p>${it.Descrição || it.Description || ''}</p>
                         <p class="prices">${price8 ? '8 fatias: '+formatBRL(parseFloat(price8)) : ''}
                            ${price4 ? ' | 4 fatias: '+formatBRL(parseFloat(price4)) : ''}</p>
                         <button class="btn" onclick="addToCart('${it.Nome || it.Name}', ${price8 || 0}, 8)">8 f</button>
                         <button class="btn" onclick="addToCart('${it.Nome || it.Name}', ${price4 || 0}, 4)">4 f</button>`;
          div.appendChild(p);
        });
        menuArea.appendChild(div);
      }
    }).catch(err => {
      menuArea.innerHTML = '<p>Erro ao carregar o cardápio. Verifique se publicou a planilha corretamente.</p>';
      console.error(err);
    });
  }

  function addToCart(name, price, size) {
    const item = cart.find(i => i.name === name && i.size === size);
    if (item) item.qty++;
    else cart.push({ name, price, size, qty: 1 });
    renderCart();
  }

  function renderCart() {
    if (!cart.length) {
      cartItemsEl.innerHTML = 'Nenhum item adicionado.';
      cartTotalEl.textContent = formatBRL(0);
      return;
    }
    cartItemsEl.innerHTML = '';
    let total = 0;
    cart.forEach(it => {
      total += it.price * it.qty;
      const div = document.createElement('div');
      div.innerHTML = `${it.qty} x ${it.name} (${it.size} fatias) — ${formatBRL(it.price * it.qty)}`;
      cartItemsEl.appendChild(div);
    });
    cartTotalEl.textContent = formatBRL(total);
  }

  document.getElementById('sendWhats').addEventListener('click', () => {
    if (!cart.length) return alert('O carrinho está vazio.');
    const name = document.getElementById('custName').value;
    const phone = document.getElementById('custPhone').value;
    const address = document.getElementById('custAddress').value;
    const pay = document.getElementById('payMethod').value;
    let msg = '*Pedido – Vitória\'s Pizzaria*%0A';
    cart.forEach(it => {
      msg += `${it.qty} x ${it.name} (${it.size} f) – ${formatBRL(it.price * it.qty)}%0A`;
    });
    msg += `%0ATotal: ${cartTotalEl.textContent}%0A%0A`;
    msg += `Nome: ${encodeURIComponent(name)}%0ATelefone: ${encodeURIComponent(phone)}%0AEndereço: ${encodeURIComponent(address)}%0APagamento: ${encodeURIComponent(pay)}`;
    window.open(`https://api.whatsapp.com/send?phone=${WHATS_NUM}&text=${msg}`, '_blank');
  });

  loadMenu();
</script>
</body>
</html>

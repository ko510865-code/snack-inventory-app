# snack-inventory-app
<!DOCTYPE html>
<html lang="my">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Snack Inventory</title>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<style>
body{font-family:Arial,sans-serif;background:#f3f5f8;margin:0;padding:16px;color:#222}
.container{max-width:750px;margin:auto}
.card{background:white;padding:18px;border-radius:12px;margin:12px 0;box-shadow:0 2px 8px #0001}
h2{margin-top:0;color:#2457a6}
input,button{box-sizing:border-box;width:100%;padding:12px;margin:6px 0;border:1px solid #ddd;border-radius:8px;font-size:16px}
button{background:#2457a6;color:white;border:0;font-weight:bold}
button.secondary{background:#555}
button.danger{background:#c0392b}
.row{display:flex;gap:8px}
.row>*{flex:1;min-width:0}
.product{border-bottom:1px solid #eee;padding:12px 0}
.small{color:#777;font-size:13px}
.hidden{display:none}
</style>
</head>
<body>
<div class="container">
<h2>📦 Snack Inventory</h2>

<div class="card" id="loginBox">
<h3>အကောင့်ဝင်ရန်</h3>
<input id="email" type="email" placeholder="Email">
<input id="password" type="password" placeholder="Password">
<button onclick="login()">Login ဝင်မည်</button>
<p id="loginMsg" class="small"></p>
</div>

<div id="app" class="hidden">
<div class="card">
<div id="userInfo"></div>
<button class="secondary" onclick="logout()">Logout ထွက်မည်</button>
</div>

<div class="card">
<h3>ပစ္စည်းရှာရန်</h3>
<input id="search" placeholder="ပစ္စည်းအမည် ရိုက်ရှာပါ..." oninput="renderProducts()">
</div>

<div class="card" id="addBox">
<h3>ပစ္စည်းအသစ်ထည့်ရန်</h3>
<input id="pname" placeholder="ပစ္စည်းအမည်">
<div class="row">
<input id="cost" type="number" placeholder="အရင်းဈေး">
<input id="price" type="number" placeholder="လက်ကားဈေး">
</div>
<div class="row">
<input id="stock" type="number" placeholder="လက်ကျန်အရေအတွက်">
<input id="unit" placeholder="ယူနစ် (ခု/ထုပ်/ဘူး)">
</div>
<button onclick="addProduct()">ပစ္စည်းထည့်မည်</button>
<p id="addMsg" class="small"></p>
</div>

<div class="card">
<h3>ပစ္စည်းစာရင်း</h3>
<div id="products">Loading...</div>
</div>
</div>
</div>

<script>
const SUPABASE_URL = "[YOUR_PROJECT_URL](https://gawqsrmxyzlmjtvdihut.supabase.co/rest/v1/)";
const SUPABASE_KEY = "sb_publishable_1VX8GMHnNCCmFF96mSqo5g_hzPhca_w";

const db = window.supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
let products = [];
let currentUser = null;
let currentRole = null;

async function login(){
  const email = document.getElementById("email").value.trim();
  const password = document.getElementById("password").value;
  const msg = document.getElementById("loginMsg");
  msg.textContent = "ဝင်နေပါသည်...";
  const {error} = await db.auth.signInWithPassword({email,password});
  if(error){msg.textContent = error.message;return;}
  await startApp();
}

async function logout(){
  await db.auth.signOut();
  location.reload();
}

async function startApp(){
  const {data:{user}} = await db.auth.getUser();
  if(!user)return;
  currentUser = user;

  const {data:profile,error} = await db.from("profiles")
    .select("display_name,role").eq("id",user.id).single();

  if(error){
    alert("Profile ဖတ်မရပါ: "+error.message);
    return;
  }

  currentRole = profile.role;
  document.getElementById("loginBox").classList.add("hidden");
  document.getElementById("app").classList.remove("hidden");
  document.getElementById("userInfo").textContent =
    (profile.display_name || user.email)+" ("+currentRole+")";

  if(currentRole !== "admin"){
    document.getElementById("addBox").classList.add("hidden");
  }
  await loadProducts();
}

async function loadProducts(){
  document.getElementById("products").textContent="Loading...";
  const {data,error} = await db.from("products")
    .select("id,name,cost_price,wholesale_price,stock,unit")
    .order("name");

  if(error){
    document.getElementById("products").textContent=error.message;
    return;
  }
  products = data || [];
  renderProducts();
}

function renderProducts(){
  const keyword = document.getElementById("search").value.toLowerCase();
  const list = products.filter(p=>p.name.toLowerCase().includes(keyword));
  const box = document.getElementById("products");

  if(!list.length){
    box.textContent="ပစ္စည်းမတွေ့ပါ။";
    return;
  }

  box.innerHTML="";
  list.forEach(p=>{
    const item=document.createElement("div");
    item.className="product";

    const title=document.createElement("b");
    title.textContent=p.name;
    item.appendChild(title);

    const details=document.createElement("p");
    details.textContent=
      "အရင်းဈေး: "+p.cost_price+
      " Ks | လက်ကားဈေး: "+p.wholesale_price+
      " Ks | လက်ကျန်: "+p.stock+" "+(p.unit||"");
    item.appendChild(details);

    if(currentRole==="admin"){
      const edit=document.createElement("button");
      edit.textContent="လက်ကျန်ပြင်ရန်";
      edit.onclick=()=>changeStock(p.id,p.stock);
      item.appendChild(edit);

      const del=document.createElement("button");
      del.className="danger";
      del.textContent="ဖျက်ရန်";
      del.onclick=()=>deleteProduct(p.id);
      item.appendChild(del);
    }
    box.appendChild(item);
  });
}

async function addProduct(){
  if(currentRole!=="admin")return;
  const name=document.getElementById("pname").value.trim();
  const cost=Number(document.getElementById("cost").value);
  const price=Number(document.getElementById("price").value);
  const stock=Number(document.getElementById("stock").value);
  const unit=document.getElementById("unit").value.trim();
  const msg=document.getElementById("addMsg");

  if(!name){msg.textContent="ပစ္စည်းအမည်ထည့်ပါ။";return;}
  if(!Number.isFinite(cost)||!Number.isFinite(price)||!Number.isFinite(stock)){
    msg.textContent="ဈေးနှုန်းနဲ့ အရေအတွက် မှန်ကန်စွာထည့်ပါ။";
    return;
  }

  const {error}=await db.from("products").insert({
    name,cost_price:cost,wholesale_price:price,stock,unit
  });
  if(error){msg.textContent=error.message;return;}

  msg.textContent="ပစ္စည်းထည့်ပြီးပါပြီ။";
  ["pname","cost","price","stock","unit"].forEach(id=>{
    document.getElementById(id).value="";
  });
  await loadProducts();
}

async function changeStock(id,oldStock){
  if(currentRole!=="admin")return;
  const value=prompt("လက်ကျန်အသစ်ထည့်ပါ",oldStock);
  if(value===null)return;
  const newStock=Number(value);
  if(!Number.isFinite(newStock)||newStock<0){
    alert("အရေအတွက်မှန်ကန်စွာထည့်ပါ။");
    return;
  }
  const {error}=await db.from("products")
    .update({stock:newStock}).eq("id",id);
  if(error){alert(error.message);return;}
  await loadProducts();
}

async function deleteProduct(id){
  if(currentRole!=="admin")return;
  if(!confirm("ဒီပစ္စည်းကို ဖျက်မှာသေချာပါသလား။"))return;
  const {error}=await db.from("products").delete().eq("id",id);
  if(error){alert(error.message);return;}
  await loadProducts();
}

db.auth.getSession().then(({data})=>{
  if(data.session)startApp();
});
</script>
</body>
</html>

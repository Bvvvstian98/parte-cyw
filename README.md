  
//MARK: Función Carrito
 
//datos del carrito y productos, en produccion esto viene de la BD
var dbUsuarios = [
  { id: 1, wishlist: [101, 103], carrito: [] },
  { id: 2, wishlist: [102, 104, 105], carrito: [] },
  { id: 3, wishlist: [], carrito: [] },
  { id: 4, wishlist: [101], carrito: [] },
  { id: 5, wishlist: [103, 107, 108], carrito: [] }
];
 
var dbProductos = [
  { id: 101, nom: "Laptop Pro 15",       prec: 1200000, stock: 5,  activo: true  },
  { id: 102, nom: "Mouse Inalambrico",   prec: 25000,   stock: 50, activo: true  },
  { id: 103, nom: "Teclado Mecanico RGB",prec: 85000,   stock: 20, activo: true  },
  { id: 104, nom: "Monitor 4K 27\"",     prec: 450000,  stock: 8,  activo: true  },
  { id: 105, nom: "Auriculares Bluetooth",prec: 75000,  stock: 30, activo: true  },
  { id: 106, nom: "Webcam HD 1080p",     prec: 45000,   stock: 15, activo: true  },
  { id: 107, nom: "SSD 1TB",             prec: 95000,   stock: 25, activo: true  },
  { id: 108, nom: "Memoria RAM 16GB",    prec: 65000,   stock: 40, activo: true  },
  { id: 109, nom: "Silla Gamer",         prec: 350000,  stock: 10, activo: false },
  { id: 110, nom: "Hub USB-C 7 en 1",    prec: 38000,   stock: 60, activo: true  }
];
 
//helpers internos, para no copiar el loop en todos lados
function buscarUsuario(userId){
  for (var i = 0; i < dbUsuarios.length; i++){
    if (dbUsuarios[i].id === userId) return dbUsuarios[i];
  }
  return null;
}
 
function buscarProducto(prodId){
  for (var i = 0; i < dbProductos.length; i++){
    if (dbProductos[i].id === prodId) return dbProductos[i];
  }
  return null;
}
 
//suma el total del carrito con los precios acutuales del producto
function calcularTotalCarrito(carrito){
  var total = 0;
  for (var i = 0; i < carrito.length; i++){
    var prod = buscarProducto(carrito[i].prodId);
    if (prod) total += prod.prec * carrito[i].qty;
  }
  return total;
}
 
//agrega un producto al carrito y si ya estaba suma la cantidad
function addCart(prodId, qty, userId){
  var usuario = buscarUsuario(userId);
  if (!usuario) return { ok: false, msg: "usuario no encontrado" };
 
  var producto = buscarProducto(prodId);
  if (!producto)        return { ok: false, msg: "producto no encontrado" };
  if (!producto.activo) return { ok: false, msg: "producto no disponible" };
  if (producto.stock < qty) return { ok: false, msg: "stock insuficiente" };
 
  var itemExistente = null;
  for (var i = 0; i < usuario.carrito.length; i++){
    if (usuario.carrito[i].prodId === prodId){
      itemExistente = usuario.carrito[i];
      break;
    }
  }
 
  if (itemExistente){
    itemExistente.qty += qty;
  } else {
    usuario.carrito.push({ prodId: prodId, qty: qty, addedAt: new Date() });
  }
 
  return{
    ok: true,
    msg: "producto agregado al carrito",
    data:{
      carrito: usuario.carrito,
      total: calcularTotalCarrito(usuario.carrito)
    }
  };
}
 
//MARK: Función Wishlist
 
//acciones de la wishlist: get para ver la lista, add para agregar y remove para sacar 
function wishlist(accion, userId, prodId){
  var usuario = buscarUsuario(userId);
  if (!usuario) return { ok: false, msg: "usuario no encontrado" };
 
  if (accion === "get"){
    return { ok: true, wishlist: usuario.wishlist };
  }
 
  if (accion === "add"){
    if (usuario.wishlist.indexOf(prodId) !== -1){
      return { ok: false, msg: "producto ya en wishlist" };
    }
    usuario.wishlist.push(prodId);
    return { ok: true, msg: "agregado a wishlist", wishlist: usuario.wishlist };
  }
 
  if (accion === "remove"){
    var idx = usuario.wishlist.indexOf(prodId);
    if (idx === -1) return { ok: false, msg: "producto no esta en wishlist" };
    usuario.wishlist.splice(idx, 1);
    return { ok: true, msg: "removido de wishlist", wishlist: usuario.wishlist };
  }
 
  return { ok: false, msg: "accion no reconocida" };
}
 
//MARK: Función Cupones
 
//lista de cupones disponibles, despues sale de la BD
var dbCupones = [
  { code: "DESC10",     tipo: "porcentaje", valor: 10,   minCompra: 50000,  maxUsos: 100,  usos: 45,  activo: true, expira: "2024-12-31", usuarios: [] },
  { code: "DESC20",     tipo: "porcentaje", valor: 20,   minCompra: 100000, maxUsos: 50,   usos: 50,  activo: true, expira: "2024-06-30", usuarios: [] },
  { code: "ENVGRATIS",  tipo: "envio",      valor: 100,  minCompra: 30000,  maxUsos: 200,  usos: 180, activo: true, expira: "2024-12-31", usuarios: [] },
  { code: "BIENVENIDO", tipo: "fijo",       valor: 5000, minCompra: 20000,  maxUsos: 1000, usos: 523, activo: true, expira: "2025-12-31", usuarios: [] },
  { code: "VIP2024",    tipo: "porcentaje", valor: 25,   minCompra: 200000, maxUsos: 20,   usos: 15,  activo: true, expira: "2024-12-31", usuarios: [1, 3, 5] }
];
 
//revisa y aplica el cupon, retorna el monto de descuento calculado segun el tipo
function cupon(code, userId, cartTotal){
  var cup = null;
  for (var i = 0; i < dbCupones.length; i++){
    if (dbCupones[i].code === code) { cup = dbCupones[i]; break; }
  }
 
  if (!cup)                                      return { ok: false, msg: "cupon no existe", descuento: 0 };
  if (!cup.activo)                               return { ok: false, msg: "cupon inactivo", descuento: 0 };
  if (new Date() > new Date(cup.expira))         return { ok: false, msg: "cupon expirado", descuento: 0 };
  if (cup.usos >= cup.maxUsos)                   return { ok: false, msg: "cupon agotado", descuento: 0 };
  if (cartTotal < cup.minCompra)                 return { ok: false, msg: "monto minimo no alcanzado", descuento: 0 };
  if (cup.usuarios.length > 0 && cup.usuarios.indexOf(userId) === -1)
                                                 return { ok: false, msg: "cupon no valido para este usuario", descuento: 0 };
 
  var descuento = 0;
  if (cup.tipo === "porcentaje") descuento = cartTotal * (cup.valor / 100);
  if (cup.tipo === "fijo")       descuento = Math.min(cup.valor, cartTotal);
  if (cup.tipo === "envio")      descuento = cup.valor;
 
  cup.usos++;
  return { ok: true, msg: "cupon aplicado", descuento: descuento, tipo: cup.tipo };
}
*/

# Servicio de Órdenes

[Volver a Inicio](../README.md)

> IMPORTANTE: Este es un ejemplo funcional de del servicio de órdenes, puedes modificarlas para que se adapten a los requerimientos de tu proyecto.

```ts
// src/services/orders.service.ts
import {
  collection,
  doc,
  getDoc,
  getDocs,
  orderBy,
  query,
  runTransaction,
  serverTimestamp,
  Timestamp,
  updateDoc,
  where,
  type DocumentData,
  type FirestoreDataConverter,
} from "firebase/firestore";
import { db } from "../config/firebase";
import type { Order, OrderStatus } from "../types/order.types";
import type { Product } from "../types/product.types";

// ======================================================
// TIPOS
// ======================================================
// Cómo se almacenan los datos en Firestore:
// - No se almacena "id" dentro del documento.
// - createdAt y updatedAt son Timestamp.
type OrderFirestore = Omit<Order, "id" | "createdAt" | "updatedAt"> & {
  createdAt: Timestamp;
  updatedAt?: Timestamp;
};

// Datos necesarios para crear una orden:
// El id y las fechas son generados por Firestore.
type CreateOrderData = Omit<Order, "id" | "createdAt" | "updatedAt" | "status">;

// ======================================================
// CONVERTER
// ======================================================
// Transformación entre Firestore y nuestra aplicación:
//
// Firestore:
// Timestamp -> fromFirestore() -> Date
//
// Aplicación:
// Date -> toFirestore() -> Timestamp
const orderConverter: FirestoreDataConverter<Order> = {
  // Antes de enviar a Firestore:
  toFirestore(order: Order): DocumentData {
    const { id, ...data } = order;
    return {
      ...data,
    };
  },

  // Recibidos desde Firestore:
  fromFirestore(snapshot) {
    const data = snapshot.data() as OrderFirestore;

    return {
      ...data,
      id: snapshot.id,
      createdAt: data.createdAt.toDate(),
      updatedAt: data.updatedAt?.toDate(),
    } satisfies Order;
  },
};

// ======================================================
// COLECCIÓN
// ======================================================
// Devuelve colección transformada por orderConverter:
function ordersCol() {
  return collection(db, "orders").withConverter(orderConverter);
}

// ======================================================
// CREAR ORDEN
// ======================================================
export async function createOrderFromCart(
  orderData: CreateOrderData,
): Promise<string> {
  // Firebase genera previamente el ID de la orden.
  //
  // La transacción utilizará esta referencia para crear
  // el documento solamente si todas las operaciones
  // finalizan correctamente.
  const orderRef = doc(collection(db, "orders"));

  await runTransaction(db, async (transaction) => {
    // Referencias de los productos involucrados.
    const productRefs = orderData.items.map((orderItem) =>
      doc(db, "products", orderItem.productId),
    );

    // 1. LEER TODOS LOS PRODUCTOS
    const productSnapshots = await Promise.all(
      productRefs.map((ref) => transaction.get(ref)),
    );

    // 2. VALIDAR STOCK
    productSnapshots.forEach((snapshot, index) => {
      if (!snapshot.exists()) {
        throw new Error(
          `Producto no encontrado: ${orderData.items[index].productId}`,
        );
      }
      const product = snapshot.data();
      const requestedQuantity = orderData.items[index].quantity;
      if (product.stock < requestedQuantity) {
        throw new Error(
          `Ya no contamos con stock suficiente para ${product.name}. ` +
            `Solo disponemos de ${product.stock} unidades.`,
        );
      }
    });

    // 3. DESCONTAR STOCK
    productSnapshots.forEach((snapshot, index) => {
      const product = snapshot.data() as Product;
      const requestedQuantity = orderData.items[index].quantity;
      transaction.update(productRefs[index], {
        stock: product.stock - requestedQuantity,
      });
    });

    // 4. CREAR ORDEN
    const orderDataToSave = {
      userId: orderData.userId,
      items: orderData.items,
      total: orderData.total,
      status: "pending" as const,
    };
    transaction.set(orderRef, {
      ...orderDataToSave,
      createdAt: serverTimestamp(),
    });
  });

  return orderRef.id;
}

// ======================================================
// OBTENER ORDEN POR ID
// ======================================================
export async function getOrderById(orderId: string): Promise<Order | null> {
  const snap = await getDoc(doc(ordersCol(), orderId));
  if (!snap.exists()) {
    return null;
  }
  return snap.data();
}

// ======================================================
// OBTENER ÓRDENES DE UN USUARIO
// ======================================================
export async function getOrdersByUser(userId: string): Promise<Order[]> {
  const q = query(
    ordersCol(),
    where("userId", "==", userId),
    orderBy("createdAt", "desc"),
  );
  const snap = await getDocs(q);
  return snap.docs.map((doc) => doc.data());
}

// ======================================================
// LISTAR ÓRDENES — ADMIN
// ======================================================
export async function listOrdersAdmin(params?: {
  status?: OrderStatus;
}): Promise<Order[]> {
  const q = params?.status
    ? query(
        ordersCol(),
        where("status", "==", params.status),
        orderBy("createdAt", "desc"),
      )
    : query(ordersCol(), orderBy("createdAt", "desc"));
  const snap = await getDocs(q);
  return snap.docs.map((doc) => doc.data());
}

// ======================================================
// ACTUALIZAR ESTADO — ADMIN
// ======================================================
export async function updateOrderStatusAdmin(
  orderId: string,
  nextStatus: OrderStatus,
): Promise<void> {
  await updateDoc(doc(ordersCol(), orderId), {
    status: nextStatus,
    updatedAt: serverTimestamp(),
  } as unknown as Partial<Order>);
}
```

---

[Volver a Inicio](../README.md)

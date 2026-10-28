const express = require('express');
const app = express();

app.use(express.json());

// Base de datos temporal en memoria
let transacciones = {};

// 1. ENDPOINT: Solicitar QR (El ESP32 llama aquí)
app.post('/api/v1/cobrar-codi', (req, res) => {
    const { id_maquina, monto } = req.body;
    const idTransaccion = "FOLIO_" + Math.floor(Math.random() * 100000);

    console.log(`[CoDi] Solicitud de máquina: ${id_maquina} por $${monto}`);

    // Cadena de texto simulada que el ESP32 convertirá en QR físico
    const cadenaCoDiSimulada = `CODI:{"v":{"CO":1,"vlo":${monto},"cr":"MXN","nc":"138180000000000000","bnc":40138,"ref":"${idTransaccion}","com":"Acceso_Bano"}}`;

    // Guardamos la transacción con su ID único como PENDIENTE
    transacciones[idTransaccion] = "PENDIENTE";

    // Para que el ESP32 sepa qué folio buscar, le regresamos el ID en las cabeceras HTTP 
    // y la cadena del QR en el cuerpo del mensaje
    res.setHeader('X-Transaction-ID', idTransaccion);
    res.status(200).send(cadenaCoDiSimulada);
});

// 2. ENDPOINT: Verificar Pago (El ESP32 hace sondeo aquí cada 3 segundos)
app.get('/api/v1/verificar-pago', (req, res) => {
    const { id_transaccion } = req.query;

    if (!id_transaccion || !transacciones[id_transaccion]) {
        return res.status(404).send("NOT_FOUND");
    }

    // Retorna "PENDIENTE" o "COMPLETADO"
    res.status(200).send(transacciones[id_transaccion]);
});

// 3. ENDPOINT DE SIMULACIÓN: Para que apruebes el pago desde tu cel o compu
app.get('/pagar', (req, res) => {
    const { id } = req.query;
    if (transacciones[id]) {
        transacciones[id] = "COMPLETADO";
        res.send(`<h1>Pago del folio ${id} aprobado exitosamente en tiempo real.</h1>`);
    } else {
        res.status(404).send("Transacción no encontrada.");
    }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Servidor CoDi corriendo en el puerto ${PORT}`);
});

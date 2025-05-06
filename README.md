# Parcial 3
<p>
En este documento se muestra los distintos métodos que se utilizaron para crear la API
</p>


######Este método GET verifica cuántos envíos tiene un usuario disponibles, cuántos ha usado, cuánto ha gastado en total en dólares y cuál es su saldo restante.

```javascript
router.get('/usuario/:usuarioId/disponibles', async (req, res) => {
    try {
        const { usuarioId } = req.params;

        const usuario = await Usuario.findById(usuarioId);
        if (!usuario) {
            return res.status(404).json({ error: 'Usuario no encontrado' });
        }

        const enviosUsados = await Envio.find({ usuarioId });

        const usados = enviosUsados.length;
        const disponibles = usuario.totalEnviosComprados - usados;

        res.json({
            usuario: usuario.nombre,
            totalComprados: usuario.totalEnviosComprados,
            usados,
            disponibles,
            envios: enviosUsados.map(envio => ({
                id: envio._id,
                producto: envio.producto,
                costo: envio.costo,
                estatus: envio.estatus
            }))
        });

    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al obtener datos del usuario' });
    }
});

```
######Este método POST agrega un nuevo ususrio a la base de datos con los siguientes campos: (nombre, dirección, teléfono, referencia, observación)

```javascript
router.post('/', async (req, res) => {
    try {
        const { nombre, direccion, telefono, referencia, observacion, creditos, plan } = req.body;

        const usuario = new Usuario({
            nombre,
            direccion,
            telefono,
            referencia,
            observacion,
            creditos,
            plan
        });

        await usuario.save();
        res.status(201).json({ message: 'Usuario registrado con éxito', usuario });
    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al registrar el usuario' });
    }
});
```
######Este metódo POST agrega más información del envio con los campos: (nombre, dirección, teléfono, referencia, observación). Tambien se modifica el credito del cliente y el precio del envio.

```javascript
router.post('/', async (req, res) => {
    try {
        const { usuarioId, descripcion, peso, bultos, fecha_entrega, observacion } = req.body;

        const usuario = await Usuario.findById(usuarioId);
        if (!usuario) return res.status(404).json({ error: 'Usuario no encontrado' });

        if (usuario.creditos <= 0) {
            return res.status(400).json({ error: 'Usuario sin créditos disponibles' });
        }

        // Calcular costo según peso
        const bloques = Math.ceil(peso / 3); // Cada 3 lb cuenta como 1 crédito
        const costoUnitario = {
            'plan30': 4.5,
            'plan40': 4.0,
            'plan60': 3.0
        }[usuario.plan] || 4.5;

        const costoFinal = bloques * costoUnitario;

        // Crear producto
        const producto = new Producto({ descripcion, peso, bultos, fecha_entrega });
        await producto.save();

        // Crear envío
        const envio = new Envio({
            usuarioId,
            productoId: producto._id,
            observacion,
            costoFinal
        });
        await envio.save();

        // Restar créditos al usuario
        usuario.creditos -= 1;
        await usuario.save();

        res.status(201).json({ message: 'Envío registrado', envio });

    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al registrar el envío' });
    }
});
```

######Con este método GET se puede consultar, con su ID, los envios realizados por un cliente.

```javascript 
router.get('/usuario/:usuarioId/disponibles', async (req, res) => {
    try {
        const { usuarioId } = req.params;

        const usuario = await Usuario.findById(usuarioId);
        if (!usuario) {
            return res.status(404).json({ error: 'Usuario no encontrado' });
        }

        const enviosUsados = await Envio.find({ usuarioId });

        const usados = enviosUsados.length;
        const disponibles = usuario.totalEnviosComprados - usados;

        res.json({
            usuario: usuario.nombre,
            totalComprados: usuario.totalEnviosComprados,
            usados,
            disponibles,
            envios: enviosUsados.map(envio => ({
                id: envio._id,
                producto: envio.producto,
                costo: envio.costo,
                estatus: envio.estatus
            }))
        });

    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al obtener datos del usuario' });
    }
});
```

######Este método DESTROY elimina un envio, luego devuelve el saldo al cliente al que se le elimino el envio.

```javascript 
router.delete('/:id', async (req, res) => {
    try {
        const envio = await Envio.findById(req.params.id);
        if (!envio) {
            return res.status(404).json({ error: 'Envío no encontrado' });
        }

        // Obtener al usuario
        const usuario = await Usuario.findById(envio.usuarioId);
        if (!usuario) {
            return res.status(404).json({ error: 'Usuario no encontrado' });
        }

        const COSTO_ENVIO = 10;

        usuario.saldo += COSTO_ENVIO;
        await usuario.save();

        // Eliminar el envío
        await envio.deleteOne();

        res.json({ message: 'Envío eliminado y saldo reembolsado al usuario' });
    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Error al eliminar el envío' });
    }
});
```

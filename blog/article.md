# Condiciones de carrera: el fallo invisible que arruinaba las reservas de pádel
En mi pueblo, teníamos una app para reservar las pistas de pádel del polideportivo municipal. 

Funcionaba bien… hasta que dos personas aparecían a la misma hora diciendo que tenían la pista reservada. 

Y ambas lo demostraban con capturas de pantalla y sus respectivos correos de reserva.

¿Resultado? 

Enfados, discusiones, críticas, partidos perdidos...

Este problema se llama condición de carrera, y aunque suena técnico, es fácil de entender… 

y más fácil todavía de sufrir si no lo prevés cuando programas.

# ¿Qué es una condición de carrera?
Imagina que tú y otra persona abrís la app al mismo tiempo. 

Veis que la pista de las 19:00 está libre.

Pulsáis “Reservar” casi a la vez. 

La app guarda los dos datos... y ambos tenéis la pista “confirmada”.

Esto sucede porque ambas operaciones leen el estado "libre" antes de que ninguna haya escrito "reservada".

Es como si dos personas vieran una silla vacía en una cafetería y se sentaran a la vez. Pero en la app, no hay camarero que medie.

# ¿Cómo se soluciona una condición de carrera?
La solución es programar con precaución y usar algo que se llama transacciones atómicas. 

Básicamente, una transacción es un “todo o nada”: o se hace todo el proceso de reserva, o no se hace nada.

Si durante ese proceso detectamos que la pista ya ha sido reservada, la operación se cancela y se avisa al usuario.

# ¿Y cómo se ve eso en código?
Aquí un ejemplo real que uso en una aplicación que desarrollé para reservar espacios comunes en comunidades de vecinos, entre ellas, pistas deportivas.

El código no es literal del que tengo implementado, ya que uso procedimientos almacenados en la base de datos para mejorar el rendimiento, pero te dejo este ejemplo para que lo puedas seguir y entender.

```import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

async function reserveHourWithTransaction(
  communityId: string,
  courtId: string,
  dayId: string,
  hourId: number,
  user: { email: string; nombre: string }
): Promise<boolean> {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // 1. Obtener la hora específica con FOR UPDATE para bloquearla
    const { rows } = await client.query(
      `SELECT * FROM hours 
       WHERE id = $1 AND day_id = $2 AND court_id = $3 AND community_id = $4 
       FOR UPDATE`,
      [hourId, dayId, courtId, communityId]
    );

    if (rows.length === 0) throw new Error('Hora no encontrada');

    const hour = rows[0];
    if (hour.user_email) throw new Error('Hora ya reservada');

    // 2. Actualizar la reserva
    await client.query(
      `UPDATE hours 
       SET user_email = $1, user_nombre = $2, reserve_type = $3 
       WHERE id = $4`,
      [user.email, user.nombre, 'reserved', hourId]
    );

    await client.query('COMMIT');
    return true;
  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Error en la reserva:', error.message);
    return false;
  } finally {
    client.release();
  }
}

```

Este código hace lo siguiente:

Accede al documento que contiene las reservas de un día.

Comprueba si la hora sigue libre.

Si está libre, la reserva en un único paso atómico.

Si alguien más ha reservado justo antes, el proceso se cancela automáticamente.

# ¿Por qué es tan importante?
Porque la confianza del usuario está en juego.

Si tu app dice que la pista está libre, debe estarlo.

Si dice que la tienes reservada, no debería haber dudas.

Y más aún si hablamos de actividades comunitarias, donde un fallo de software puede convertirse en un problema entre vecinos.

# Conclusión: cuando programar bien evita conflictos reales
La programación no es solo escribir código bonito o rápido. 

Es prever situaciones reales.

Evitar condiciones de carrera en una app de reservas es esencial.

Tengo desplegada mi aplicación desde 2021 en varias comunidades y hasta la fecha ni una reserva duplicada.

¿Te ha pasado algo parecido? 

¿Tienes una app que gestiona reservas? 

Escríbeme y te ayudo a prevenir estos errores invisibles.
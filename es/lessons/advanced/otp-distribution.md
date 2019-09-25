---
version: 1.0.1
title: OTP Distribution
---

## Introducción a Distrubición
Podemos correr nuestras aplicaciones de elixir en un conjunto de nodos distribuidos en uno o multiples servidores.

Elixir nos permite comunicarnos a través de estos nodos por medio de distintos mecanismos que esbozaremos en esta lección.

{% include toc.html %}

## Comunicación Entre Nodos

Elixir se ejecuta en la máquina virtual (VM) de Erlang, lo que significa que tiene acceso a todo el poder de la [funcionalidad distribuida](http://erlang.org/doc/reference_manual/distributed.html) de Erlang.

> Un sistema distribuida de Erlang consiste en un numero de systemas de ejecución en tiempo real que se comunican entre si.
Cada sistema de ejecución (runntime) es llamado un nodo.

Un node es cualquier systema de ejecución al cual se le ha dado un nombre.
Podemos iniciar un nodo al iniciar una sesión en `iex` y nombrarla.

```bash
iex --sname alex@localhost
iex(alex@localhost)>
```

Ahora iniciemos otro nodo en otra ventana de terminal:

```bash
iex --sname kate@localhost
iex(kate@localhost)>
```

Estos dos nodos pueden enviarse mensajes entre ellos utilizando `Node.spawn_link/2`.

### Comunicación con `Node.spawn_link/2`

Esta función toma dos argumentos:
* El nombre del nodo al que deseas conectarte
* La función a ser ejecutada por el proceso remoto ejecutado en ese nodo

Esta establece la conexión al nodo remoto y ejecuta la función dada en el nodo, retornando el PID del proceso enlazado.

Ahora definamos un módulo, `Kate`, en el nodo `kate` que sabe como presentar a Kate, la persona:

```elixir
iex(kate@localhost)> defmodule Kate do
...(kate@localhost)>   def say_name do
...(kate@localhost)>     IO.puts "Hi, my name is Kate"
...(kate@localhost)>   end
...(kate@localhost)> end
```

#### Envio de Mensajes

Ahora, podemos utilizar [`Node.spawn_link/2`](https://hexdocs.pm/elixir/Node.html#spawn_link/2) para que el nodo `alex` requiera al nodo `kate` llamar a la función `say_name/0`:

```elixir
iex(alex@localhost)> Node.spawn_link(:kate@localhost, fn -> Kate.say_name end)
Hi, my name is Kate
#PID<10507.132.0>
```

#### Una Nota acerca de I/O y Nodos

Tienes que notar que, a pesar de que `Kate.say_name/0` se ejecuta en el nodo remoto, es el local, o el que hace la llamada, el que recive la salida de `IO.puts`.
Esto es porque el nodo local es el **lider del grupo**.
La máquina virtual de Erlang maneja las operaciones de entrada y salida (I/O) por medio de procesos.
Esto permite ejecutar operaciones de entrada y salida I/O, como `IO.puts`, a través de nodos distribuidos.
Estos procesos distribuidos son administrados por el proceso de entrada y salida I/O del lider del grupo.
El lider del grupo es siempre el nodo que hace spawn al proceso.
De manera que, como nuestro nodo `alex` es desde el que ejecutamos la llamada `spawn_link/2`, ese nodo es el líder del grupo y la salida de `IO.puts` será dirigida a la transimisión de la salida standard de ese nodo.

#### Respondiendo a Mensajes

¿Qué sucede si queremos que el nodo que recive los mensajes envíe alguna *respuesta* de regreso al transmisor? Podemos utilizar un simple `receive/1` y la configuración de [`send/3`](https://hexdocs.pm/elixir/Process.html#send/3) para lograr exactamente eso.

Tendremos nuestro nodo `alex` con un spawn enlazado al nodo `kate` y dar al nodo `kate` una función anónima a ejecutar.
Esa función anónima escuchará la recepción de una tupla particular que describa un mensaje y el PID del nodo `alex`.

```elixir
iex(alex@localhost)> pid = Node.spawn_link :kate@localhost, fn ->
...(alex@localhost)>   receive do
...(alex@localhost)>     {:hi, alex_node_pid} -> send alex_node_pid, :sup?
...(alex@localhost)>   end
...(alex@localhost)> end
#PID<10467.112.0>
iex(alex@localhost)> pid
#PID<10467.112.0>
iex(alex@localhost)> send(pid, {:hi, self()})
{:hi, #PID<0.106.0>}
iex(alex@localhost)> flush()
:sup?
:ok
```

#### Una Nota acerca de Comunicación entre Nodos en Redes Distintas

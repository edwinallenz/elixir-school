---
version: 1.0.1
title: OTP Distribution
---

## Introducción a Distribución
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

Tendremos nuestro nodo `alex` que genera un proceso enlazado al nodo `kate` y dar al nodo `kate` una función anónima a ejecutar.
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

Si deseas enviar mensajes entre nodos en distintas redes, es necesario iniciar los nodos nombrados con un cookie compartido.

```bash
iex --sname alex@localhost --cookie secret_token
```

```bash
iex -sname kate@localhost --cookie secret_token
```

Solo los nodos que se inicien con el mismo `cookie` podrán conectarse satisfactoriamente entre si.

#### Limitaciones de `Node.spawn_link/1`

Mientras que `Node.spawn_link/2` ilustra las relaciones entre nodos y la manera en que estos pueden enviar mensajes entre ellos, _no_ es necesariamente la decisión correcta para una aplicación que se ejecutará por medio de nodos distribuidos.
`Node.spawn_link/2` genera proceso de forma aislada, es decir procesos que no son supervisados.
Si solo hubiera una manera de generar procesos supervisados y asíncronos _entre nodos_...

## Tareas Distribuidas
[Tareas distribuidas](https://hexdocs.pm/elixir/master/Task.html#module-distributed-tasks) nos permiten generar tareas supervidas a traves de distintos nodos.
Vamos a construir un una aplicación supervisada sencilla que aprovecha las tareas distribuidas para permitir que usuarios puedan chatear entre ellos utilizando una sessión `iex`, a través de nodos distribuidos.

### Definiendo la Aplicación Supervisada

Genera tu aplicación:

```bash
mix new chat --sup
```

### Agregar el Supervisor de Tareas al Árbol de Supervisión

Un Supervisor de Tareas supervisará tareas dinámicamente.
Este es iniciado sin hijos, a menudo _debajo_ de un supervisor propio, y que puede ser utilizado posteriormente para supervisar cualquier número de tareas.

Agregaremos una Supervisor de Tareas al árbol de supervisión de nuestra applicación, nombralo `Chat.TaskSupervisor`.

```elixir
# lib/chat/application.ex
defmodule Chat.Application do
  @moduledoc false

  use Application

  def start(_type, _args) do
    children = [
      {Task.Supervisor, name: Chat.TaskSupervisor}
    ]

    opts = [strategy: :one_for_one, name: Chat.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

Ahora sabemos que en cualquier nodo que nuestra applicación inicie, el supervisor `Chat.Supervisor` estará corriendo y listo para supervisar tareas.

### Enviando Mensajes con Tareas Supervisadas

Vamos iniciar tareas supervisadas con la función [`Task.Supervisor.async/5`](https://hexdocs.pm/elixir/master/Task.Supervisor.html#async/5).

Esta función necesita cuatro argumentos:

* El supervisor que vamos a utilizar para supervisar la tarea.
Esto puede ser pasado como una tupla `{SupervisorName, remote_node_name}` para supervisar la tarea en el nodo remoto.
* El nombre del módulo del cual queremos ejecutar la función.
* El nombre de la función que queremos ejecutar.
* Cualquier argumentos que necesitan ser provistos para la función.

Puedes pasar un quinto, argumento opcional que describe las opciones de apagado.
Pero no nos vamos a preocupar de eso aquí.

Nuestra aplicación de Chat es muy sencilla.
Envía mensajes a nodos remotos y los nodos remotos responden a estos mensajes haciendo un `IO.puts` al STDOUT del nodo remoto.

Primero, definamos una función, `Chat.receive_message/1`, que queremos que nuestra tarea ejecute en un nodo remoto.

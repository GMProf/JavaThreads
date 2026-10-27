# Códigos completos dos exemplos

## Arquivo: ProcessoThreadDemo.java

```java
public class ProcessoThreadDemo {
    private static void identificar() {
        System.out.printf("PID=%d | thread=%s%n",
                ProcessHandle.current().pid(),
                Thread.currentThread().getName());
    }

    public static void main(String[] args) throws InterruptedException {
        identificar();
        Runnable tarefa = ProcessoThreadDemo::identificar;
        Thread trabalhadora = new Thread(tarefa, "trabalhadora");
        System.out.println("Chamada direta de run():");
        trabalhadora.run(); // Chamada comum, executada pela main.
        System.out.println("Chamada de start():");
        trabalhadora.start(); // Agenda uma nova thread de plataforma.
        trabalhadora.join(); // A main aguarda a trabalhadora terminar.
        System.out.println("Estado final: " + trabalhadora.getState());
    }
}
```

## Arquivo: EstadosThreadDemo.java

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.locks.LockSupport;

public class EstadosThreadDemo {
    // Observacao didatica com limite de tempo. Nao usar getState como
    // mecanismo de coordenacao de uma aplicacao real.
    private static void observar(Thread t, Thread.State esperado)
            throws InterruptedException {
        long inicio = System.nanoTime();
        while (t.getState() != esperado) {
            if (System.nanoTime() - inicio > TimeUnit.SECONDS.toNanos(5)) {
                throw new IllegalStateException("Nao observado: " + esperado
                        + "; estado atual: " + t.getState());
            }
            Thread.sleep(1);
        }
        System.out.println(esperado);
    }

    public static void main(String[] args) throws InterruptedException {
        Object monitor = new Object();
        CountDownLatch liberarEspera = new CountDownLatch(1);
        AtomicBoolean manterPausa = new AtomicBoolean(true);
        AtomicBoolean encerrar = new AtomicBoolean(false);
        Thread t = new Thread(() -> {
            synchronized (monitor) {
                // Apenas para tornar observavel a disputa pelo monitor.
            }
            try {
                liberarEspera.await(); // WAITING, sem prazo.
                while (manterPausa.get() && !Thread.currentThread().isInterrupted()) {
                    LockSupport.parkNanos(TimeUnit.SECONDS.toNanos(1));
                }
                // Ocupacao de CPU curta, apenas para observar RUNNABLE.
                while (!encerrar.get() && !Thread.currentThread().isInterrupted()) {
                    Thread.onSpinWait();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "observada");

        try {
            observar(t, Thread.State.NEW);
            synchronized (monitor) {
                t.start();
                observar(t, Thread.State.BLOCKED);
            }
            observar(t, Thread.State.WAITING);
            liberarEspera.countDown();
            observar(t, Thread.State.TIMED_WAITING);
            manterPausa.set(false);
            LockSupport.unpark(t);
            observar(t, Thread.State.RUNNABLE);
            encerrar.set(true);
            t.join();
            observar(t, Thread.State.TERMINATED);
        } finally {
            // Tambem encerra a demonstracao se uma observacao falhar.
            encerrar.set(true);
            manterPausa.set(false);
            liberarEspera.countDown();
            t.interrupt();
            LockSupport.unpark(t);
            t.join(2_000);
            if (t.isAlive()) {
                throw new IllegalStateException("A thread nao terminou.");
            }
        }
    }
}
```

## Arquivo: PrioridadesDemo.java

```java
public class PrioridadesDemo {
    private static void calcular() {
        long soma = 0;
        for (int i = 0; i < 5_000_000; i++) {
            soma += i;
        }
        Thread atual = Thread.currentThread();
        System.out.printf("%s | prioridade=%d | soma=%d%n",
                atual.getName(), atual.getPriority(), soma);
    }

    public static void main(String[] args) throws InterruptedException {
        Thread baixa = new Thread(PrioridadesDemo::calcular, "baixa");
        Thread alta = new Thread(PrioridadesDemo::calcular, "alta");
        baixa.setPriority(Thread.MIN_PRIORITY);
        alta.setPriority(Thread.MAX_PRIORITY);
        System.out.printf("Prioridades efetivas: baixa=%d, alta=%d%n",
                baixa.getPriority(), alta.getPriority());
        baixa.start();
        alta.start();
        baixa.join();
        alta.join();
        System.out.println("A prioridade nao garante a ordem de conclusao.");
    }
}
```

## Arquivo: ServidorSimulado.java

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.Executors;

public class ServidorSimulado {
    private static String atender(int id) throws InterruptedException {
        Thread.sleep(200); // Simula espera de I/O, sem rede real.
        return "Pedido " + id + " atendido";
    }

    public static void main(String[] args) throws Exception {
        long inicio = System.nanoTime();
        for (int id = 1; id <= 4; id++) {
            System.out.println(atender(id));
        }
        long sequencial = System.nanoTime() - inicio;

        List<Callable<String>> tarefas = new ArrayList<>();
        for (int id = 1; id <= 4; id++) {
            final int pedido = id;
            tarefas.add(() -> atender(pedido));
        }
        inicio = System.nanoTime();
        // O fechamento do executor neste formato exige Java 19+.
        try (var pool = Executors.newFixedThreadPool(2)) {
            var resultados = pool.invokeAll(tarefas);
            for (var resultado : resultados) {
                System.out.println(resultado.get());
            }
        }
        long concorrente = System.nanoTime() - inicio;
        System.out.printf("Sequencial: %.1f ms%n", sequencial / 1_000_000.0);
        System.out.printf("Pool com 2 threads: %.1f ms%n", concorrente / 1_000_000.0);
        System.out.println("Tempos ilustrativos. Nao e um benchmark.");
    }
}
```

# Procesos
- Context switch: Operation mas cara, ejecución simultanea de programas
- Optimizar programas: Stack, Heap, Multiple CPU
- Espera de los recursos: Que pasa cuando hay un fallo en esos recursos externos. 
# Threads
Forma de paralelizar un proceso en ejecución sin crear otros procesos "Lightweight processes" 
## Multi-Thread Program
- Mas de un punto de ejecución
- Multiples program counters (PCs) 
- Todos comparten un mismo espacio de direcciones
- Si un thread falla, el thread que lo creo deberia menjarlo.
## Data races
- Definir seccion critica para poder evitarlos
- Considera todo el procedimiento como una intruccion 
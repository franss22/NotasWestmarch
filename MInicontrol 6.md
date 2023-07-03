# A
La inteligencia artificial es un concepto mas amplio que el machine learning. Inteligencia artificial se refiere a técnicas para intentar replicar el comportamiento (y la inteligencia) humana, en general para intentar automatizar tareas intelectuales. Machine, por otro lado, es un tipo (o enfoque) de inteligencia artificial que consiste en tener un programa que a partir de una gran cantidad de datos iniciales, intenta modelar un comportamiento, encontrando la "regla" que relaciona los datos iniciales para luego buscar casos que cumplan esa regla en datos que se le entreguen a futuro.

# B
Para este algoritmo lo que mas se acomoda al uso es un programa que inicialmente,recia fotografías de los 10 productos. Luego, a partir de esas imágenes, genera y guarda local features para cada imagen de producto. 
Finalmente, cuando un usuario entrega una fotografía, el algoritmo calcula las local feature de la fotografía y busca matches en las imagenes de las fotos originales. Finalmente, entrega el producto de cuya foto tuvo mejores matches como candidato de producto.

# C
Para este algoritmo se toman imagenes de todos los productos, y se les taggea manualmente con sus categorías. A partir de esto, se entrena una red neuronal para que guarde deep features para cada objeto, y se usa una algoritmo de clustering para crear categorías a partir de los vectores de deep feature
Finalmente, cuando un usuario mande una nueva imagen, la red calcula las deep features de la imagen. Con este descriptor, se puede determinar a cual cluster pertenece la nueva imagen, y entregar esto como candidato.
# D
Este algoritmo funcionaría de forma similar al anterior, con la diferencia de que se debe definir un threshold de distancia. Si una imagen se le calcula una deep feature que está mas allá del threshold de distancia de todos los centroides del clustering, se señala que se debe crear una nueva categoría. Al crear una nueva categoría, se vuelve a calcular el clustering, esta vez con un nuevo centroide en el nuevo producto.


# E

# F
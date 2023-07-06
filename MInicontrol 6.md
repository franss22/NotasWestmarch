# A
La inteligencia artificial es un concepto mas amplio que el machine learning. Inteligencia artificial se refiere a técnicas para intentar replicar el comportamiento (y la inteligencia) humana, en general para intentar automatizar tareas intelectuales. Machine, por otro lado, es un tipo (o enfoque) de inteligencia artificial que consiste en tener un programa que a partir de una gran cantidad de datos iniciales, intenta modelar un comportamiento, encontrando la "regla" que relaciona los datos iniciales para luego buscar casos que cumplan esa regla en datos que se le entreguen a futuro.

# B
Para este algoritmo lo que mas se acomoda al uso es un programa que inicialmente,recia fotografías de los 10 productos. Luego, a partir de esas imágenes, genera y guarda local features para cada imagen de producto. 
Finalmente, cuando un usuario entrega una fotografía, el algoritmo calcula las local feature de la fotografía y busca matches en las imagenes de las fotos originales. Finalmente, entrega el producto de cuya foto tuvo mejores matches como candidato de producto.

# C
Para este algoritmo se toman imagenes de todos los productos, y se les taggea manualmente con sus categorías. A partir de esto, se entrena una red neuronal para que guarde deep features para cada objeto, y se usa una algoritmo de clustering (k-means) para crear categorías a partir de los vectores de deep feature
Finalmente, cuando un usuario mande una nueva imagen, la red calcula las deep features de la imagen. Con este descriptor, se puede determinar a cual cluster pertenece la nueva imagen, y entregar esto como candidato.

# D
Este algoritmo funcionaría de forma similar al anterior, con la diferencia de que se debe definir un threshold de distancia. Si una imagen se le calcula una deep feature que está mas allá del threshold de distancia de todos los centroides del clustering, se señala que se debe crear una nueva categoría. Al crear una nueva categoría, se vuelve a calcular el clustering, esta vez con un nuevo centroide en el nuevo producto.

# E
Para este algoritmo se entrena una red neuronal convolucional de clasificación de texto. El set de entrenamiento serán los producto ya existentes. Sin embargo, puede ocurrir que este set sea demasiado pequeño y por lo tanto sea necesario agrandarlo artificialmente. Una vez entrenado, la red neuronal, al entregarle un nuevo producto, entregará una lista ordenada de loas categorías a las que mas probablemente pertenece el nuevo producto.
# F
Una manera simple de hacer esto es tener 2 redes neuronales separadas, una que clasifique el texto, y otra que clasifique la imagen. Luego, multiplicamos las probabildades de ambas, para hacer un ranking hibrido. Por ejemplo, si la red neuronal para texto dice que "salsa de tomate" tiene 80% de probabilidades de pertenecer a la categoría "comida" y 20% de probabilidades de pertenecer a la categoría "aseo", y la red neuronal para imagenes indica que tiene un 50% de probabiliades de ser "comida" y 25% de probabilidades de ser "herramientas", quedfaría que la probabilidad de ser comida es 40%, la de ser aseo es 0% y la de ser herramientas es 0%.
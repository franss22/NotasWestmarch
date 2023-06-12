# P1
## a
Sea $H_K(X||Y) = E_X(Y)\oplus X$
Dado que $E_X$ es PRF, conocemos $E_X^{-1}$
luego podemos hacer al adversario $A$ tal que:

Adversario $A$:
	$a = 0^n$
	$b = 1^n$
	$y_a = E_b^{-1}(a)$
	$y_b = E_a^{-1}(b)$ 
	Return ($a||y_b, b||y_a$)

Esto da una colisión, pues 
$H_K(a||y_b) = E_a(y_b)\oplus a = E_a(E_a^{-1}(b))\oplus a = b \oplus a$
y
$H_K(b||y_a) = E_b(y_a)\oplus b = E_b(E_b^{-1}(a))\oplus b = a \oplus b$
con
$a||y_b \ne b||y_a$ y $a\oplus b = b\oplus a$ 

## b
No es seguro contra ataque de texto escogido, pues podemos hacer lo siguiente:
Sea "Mitad$_i$($x$)" la función que entrega la $i$-esima mitad de los bits de $x$
Adversario $A$:
	$a = 0^n$
	$b = 1^n$
	$T_a =$ Tag($a||a$)
	$T_b =$ Tag($b||b$)
	$F_a$ = Mitad$_1$($T_a$)
	$F_b$ = Mitad$_2$($T_b$)
	return ($a||b, F_a||F_b$)

Tenemos que 
$T_a = F_K(0||a)||F_k(1||a)$
y
$T_b = F_K(0||b)||F_k(1||b)$

Luego
$F_a = F_K(0||a)$
y
$F_b = F_k(1||b)$

Finalmente, tenemos que 
$T_K(a||b) = F_K(0||a)||F_k(1||b) = F_a||F_b$
y $a||b \notin \{a||a, b||b\}$
por lo que entregamos un par menasje, tag correcto y que no ha sido revisado antes, con 2 preguntas al oráculo, en tiempo $O(1)$.

Luego, $T_K$ no es seguro ante un ataque de mensaje escogido.

# P2

Veremos primero si $\mathcal{SE} \textrm{ es IND-CPA} \implies \overline{\mathcal{SE}} \textrm{ es IND-CPA}$, domestranbdolo por contradicción.

Sea $\overline{A}$ un adversario con ventaja significativa para $\overline{\mathcal{SE}}$ en el sentido IND-CPA. (Es decir, asumimos que $\overline{\mathcal{SE}} \textrm{ no es IND-CPA}$). Veremos que esto implica que $\mathcal{SE}$ tampoco es $\textrm{IND-CPA}$


Podemos construir un adversario $A$ contra $\mathcal{SE}$ tal que

$A$
	$k \overset{{\scriptscriptstyle\$}}{\leftarrow} \mathcal{K}$
	Correr el adversario $\overline{A}$ contestando sus preguntas tal que:
		Cuando $\overline{A}$ hace la pregunta LR($m_0, m_1$), calcular:
		$c = LR_{\mathcal{SE}}(m_0, m_1)$
		$t \overset{{\scriptscriptstyle\$}}{\leftarrow} \mathcal{T}_k(c)$
		entregamos $(c, t)$ a $\overline{A}$
		hasta que $\overline{A}$ se detiene entrega bit $b$
	return $b$

Tenemos que 
$$
\begin{split}
Adv_{A}&=|Pr[Izq_{\mathcal{SE}}^{A}\implies 1]-Pr[Der_{\mathcal{SE}}^{A}\implies 1]| \\&= |Pr[Izq_{\mathcal{\overline{SE}}}^{\overline{A}}\implies 1]-Pr[Der_{\mathcal{\overline{SE}}}^{\overline{A}}\implies 1]|\\ &=Adv_{\overline{A}}

\end{split}$$

Luego si $\overline{A}$ tiene ventaja significativa, $A$ también, por lo que demostramos la contrareciproca (y por lo tanto el postulado original).

Ahora queremos demostrar que $\mathcal{MA} \textrm{ es SUF-CMA} \implies \overline{\mathcal{SE}} \textrm{ es INT-CTXT}$
Lo haremos por contradicción:
Sea $\overline{B}$ un adversario contra $\overline{\mathcal{SE}}$ en el sentido $\textrm{INT-CTXT}$ con ventaja significativa.

Podemos construir un adversario $B$ contra $\mathcal{MA}$ en el sentido $\textrm{SUF-CMA}$ tal que:
$B$
	$k \overset{{\scriptscriptstyle\$}}{\leftarrow} \mathcal{K}$
	Correr el adversario $\overline{B}$ contestando sus preguntas tal que:
		Cuando $\overline{B}$ hace la pregunta Enc($M$), calcular:
		$c = \mathcal{E}_k(M)$
		$t =$ Tag($c$)
		entregamos $(c, t)$ a $\overline{B}$
		hasta que $\overline{B}$ se detiene y entrega $(c', t')$
	return $(c', t')$

Si $\overline{B}$ gana, significa que entregó un par $(c', t')$ tal que $c'$ no se repite, y dado $\overline{\mathcal{D}}_K$, $\overline{\mathcal{D}}_K(c', t')$ entrega un $M$ (es decir que el tag estaba bien puesto.)
Luego, 
$$
\begin{split}

Adv(B) &= Pr[SUFCMA^A_{\mathcal{MA}}\implies true]\\
&= Pr[INTCTXT^\overline{B}_{\mathcal{\overline{SE}}}\implies true] = Adv(\overline{B})
\end{split}
$$
es decir $Adv(B) = Adv(\overline{B})$, luego si $\overline{B}$ tiene ventaja significativa ($\overline{\mathcal{SE}}$ no es INT-CTXT) implica $\mathcal{MA}$  no es SUF-CMA. Demostramos la contrareciproca y por lo tanto tenemos que:

$$
\begin{split}
(\mathcal{SE} \textrm{ es IND-CPA} \implies \overline{\mathcal{SE}} \textrm{ es IND-CPA}
&\land 
\mathcal{MA} \textrm{ es SUF-CMA} \implies \overline{\mathcal{SE}} \textrm{ es INT-CTXT})\\
&\implies\\

\mathcal{SE} \textrm{ es IND-CPA} \land \mathcal{MA} \textrm{ es SUF-CMA} &\implies \overline{\mathcal{SE}} \textrm{ es IND-CPA} \land\overline{\mathcal{SE}} \textrm{ es INT-CTXT}

\end{split}
$$

# P3

Hacemos lo siguiente:



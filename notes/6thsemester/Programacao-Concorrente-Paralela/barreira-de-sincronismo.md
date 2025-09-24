# Barreira de Sincronismo

Barreiras de Sincronismo funcionam como um ponto de chegada limite para todas as threads que estão trabalhando em um mesmo workload, de modo que nenhuma delas consegue ultrapassar esse ponto até que todas tenham passado em conjunto. Deste modo, existem diversas implementações, e python por exemplo tem a sua própria classe definida nativamente.

### Implementação por contador compartilhado

Uma ideia pode ser a implementação por contador, com await. Porém essa implementação tem sérios problemas de race condition e de reset para o próximo workload, que simplesmente não existe.
```C
arrive = 0;

Process Workers [1 to N]{
    // O que quer que seja que o processo faz
    while(True){
        <arrive ++;>
        <await(arrive==N);>
    }
}
```

### Implementação por controle de Flags

Neste contexto flags de sinalização são utilizadas, de modo que existem vetores de flags com N posições (onde N é o número de processos), no qual arrive[N] é o vetor de sinalização de "chegou na barreira", e continue[N] é o vetor de sinalização de "pode continuar" a partir da barreira.
Assim, existem processos coordenadores e processos trabalhadors, de modo que o processo que espera por uma flag, deve ser o mesmo a resetá-la, e a flag não pode ser sinalizada sem ter sido limpa antes.

```C
process Worker[i=1 to N]{
	while(true){
		funcao();
		arrive[i] = 1;
		while (continue[i] == 0) skip;
		continue[i] = 0;
        //<await(continue[i]==1) continue[i]=0;>
	}
}

process Coordinator[i=1 to N]{
	while(true){
		for(int j = 1; j <= N; j++){
			while(arrive[j]==0) skip;
			arrive[j]=0
            //<await(arrive[j]==1) arrive[j]=0;>
		}
		for(int j = 1; j <= N; j++)
			continue[j]=1;
	}
}
```
### Solução com barreira de Árvore 
Essa é uma das piores soluções, já que sua implementação é mais complexa e você deixa grande parte das threads ociosas enquanto as outras soluções você tem N threads trabalhando em N processos, enquanto nessa abordagem apenas os nó folha trabalham.

![alt text](images/image-3.png)

```C

process Worker[i=1 to N]{
	while(true){
		//Código
		BARREIRA
	}
}

process Leaf{
	BARREIRA: arrive[i]=1;
	<await (continue[i]==1);>
	continue[i]=0;
}

process Root{
	BARREIRA: <await (arrive[esq]==1)   arrive[esq]=0;>
		<await (arrive[dir]==1)   arrive[dir]=0;>
		continue[esq]=1;
		continue[dir]=1;
}

process Inter{
	BARREIRA: <await (arrive[esq]==1)   arrive[esq]=0;>
		<await (arrive[dir]==1)   arrive[dir]=0;>
		arrive[i]=1;
		await(continue[i]==1)   continue[i]=0;
		continue[esq]=1;
		continue[dir]=1;
}
```
O nó folha atualiza o seu próprio arrive pq não tme filhos, e espera até que o pai dele o libere, quando isso contece ele reseta o seu continue.

O nó raiz, como não tem pai, foca o arrive dos filhos, de modo a esperar os filhos sinalizarem que chegou e resetá-las, propagando que elas podem continuar.

O nó interno espera o arrive de seus filhos, igual a raiz, sinaliza o seu próprio arrive para o nó pai dele, e espera o pai sinalizar que ele pode continuar, e reseta o seu próprio continue, e propaga para os filhos que eles podem continuar. 

### Solução de barreiras com Simetria

Existem duas barreiras por simetria, dentre elas:

## Barreia Borboleta

Esta barreira funciona apenas quando o número de processos é 2<sup>N</sup>, visto que cada processo se comunica com o seu vizinho, alterando ao longo das etapas. A comunicação entre eles é feita através de código gray. **(LER NO LIVRO)**
Segue exemplo:
![alt text](images/image-4.png)


### Barreira por Disseminação
Código de implementação:

```C
BARREIRA:
	for(s=1 to num_estagios){
		arrive[i]=arrive[i]+1
		//determinar o vizinho no estágio
		while(arrive[j]<arrive[i]){
			skip;
		}
	}
```

**LER A PORRA DO LIVRO.**
![alt text](images/image-5.png)

### Implementação por semáforo
Nessa implementação, estudar amanhã.

```C
BARREIRA:	
	for(s=1 to num_estagios){
		j = paraquemenvia(i,s);
		V(s[i][s]);
		P(s[j][s]);
	}
```

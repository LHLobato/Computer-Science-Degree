# Múltiplos Leitores e Escritores

O problema dos múltiplos leitores e múltiplos escritores é um desafio clássico de sincronização em sistemas operacionais. Ele descreve o acesso concorrente a um recurso compartilhado, como um arquivo ou um banco de dados, por dois tipos de processos: leitores, que apenas consultam o recurso sem modificá-lo, e escritores, que o modificam.

O objetivo é permitir a máxima concorrência possível, respeitando regras cruciais para a integridade dos dados. Essas regras são:

    Múltiplos leitores podem acessar o recurso ao mesmo tempo.

    Apenas um escritor pode acessar o recurso de cada vez.

    Quando um escritor está modificando o recurso, nenhum leitor ou outro escritor pode acessá-lo.

A principal dificuldade está em desenvolver um mecanismo que gerencie essa interação sem causar problemas como a inanição (starvation), onde um tipo de processo (por exemplo, os escritores) nunca consegue acesso ao recurso porque os outros (os leitores) estão sempre presentes.


int nr = 0;
sem rw = 1;

process Writer [j = 1 to N]{
	while(true){
		P(rw);
		escreve;
		V(rw);
	}
}

process Reader [i = 1 to N]{
	while(true){
		<nr = nr + 1;
		if(nr == 1) 
			P(rw);
		>
		leitura;
		< nr = nr - 1;
			if(nr == 0)
				V(rw);
		>
	}
}

o await não existe de verdade, sendo apenas uma abstração, logo, a implementação real é feita através do uso de um mutex, semáforo nesse caso.

int nr = 0;
sem rw = 1;
sem mutex = 1;

process Writer [j = 1 to N]{
	while(true){
		P(rw);
		escreve;
		V(rw);
	}
}

process Reader [i = 1 to N]{
	while(true){
		P(mutex);
		nr = nr + 1;
		if(nr == 1) 
			P(rw);
		V(mutex);
		leitura;
		P(mutex);
		nr = nr - 1;
			if(nr == 0)
				V(rw);
		V(mutex);
	}
}

Nessa solução existe o problema de starvation. Ao bloquear novos leitores quando uma requisição de escrita é executada, esse problema pode ser resolvido.


int nr = 0;
int nw = 0;
int dr = 0;
int dw = 0;
sem r = 0;
sem w = 0;
sem e = 1;

process Writer [j = 1 to N]{
	while(true){
		< await (nr == 0 and nw == 0)
			nw = nw + 1;
		>
		escreve
		< nw = nw - 1;
			SIGNAL
		>
	}
}

process Reader [i = 1 to N]{
	while(true){
		<await (nw == 0) nr = nr+1; >
		leitura;
		< nr = nr - 1;
			SIGNAL;
		>
	}
}

SIGNAL: 
	if(nw == 0 and dr > 0){
		dr = dr - 1;
		V(r);
	}
	else if(nr == 0 and nw == 0 and dw > 0){
		dw = dw - 1;
		V(w);
	}
	else
		V(e)


Por conta do uso do await, é necessário a alteração como fora visto anteriormente. Além disso, SIGNAL está sendo usado de forma errônea.


int nr = 0;
int nw = 0;
int dr = 0;
int dw = 0;
sem r = 0;
sem w = 0;
sem e = 1;

process Writer [j = 1 to N]{
	while(true){
		P(e);
		if(nr > 0 or nw > 0){
			dw = dw + 1;
			V(e);
			P(w);
		}
		nw = nw + 1;
		V(e);
		escreve;
		P(e);
		nw = nw - 1;
		if(dr > 0){
			dr = dr - 1;
			V(r);
		}
		else if(dw > 0){
			dw = dw - 1;
			V(w);
		else
			V(e);
	    }
    }
}
process Reader [i = 1 to N]{
	while(true){
		P(e)
		if(nw > 0){
			dr = dr + 1;
			V(e);
			P(r);
		}
		nr = nr + 1;
		if( dr > 0){
			dr = dr - 1;
			V(r);
		}
		else
			V(e);
		leitura;
		P(e)
		nr = nr - 1;
		if(nr == 0 and dw > 0){
			dw = dw - 1;
			V(w);
		}
		else
			V(e);
	}
}


Seria possível simplificar essa solução através do uso de monitores

monitor RW Controller{
	int nr = 0;
	int nw = 0;
	cond OKtoread, OKtowrite;
	procedure request_read(){
		while(nw > 0)
			wait(OKtoread);
		nr= nr + 1;
	}
	procedure release_read(){
		nr = nr - 1;
		if(nr == 0)
			signal(OKtowrite)
	}
	procedure request_write(){
		while(nr > 0 || nw > 0)
			wait(OKtowrite);
		nw = nw + 1;
	}
	procedure release_write(){
		nw = nw - 1;
		signal(OKtowrite);
		signalall(OKtoread);
	}
}

essa solução ainda causa starvation no escritor, mas poderia ser resolvida com duas flags.

monitor RW_Controller_Prioridade_Escritor {
    // nr = número de leitores ativos
    // nw = número de escritores ativos
    // writer_waiting = flag para indicar se há um escritor esperando
    int nr = 0;
    int nw = 0;
    bool writer_waiting = false;
    
    // Variáveis de condição
    cond OKtoread;
    cond OKtowrite;

    // --- Rotinas do Leitor ---
    procedure request_read() {
        // Se houver um escritor ativo, espera
        // Se houver um escritor esperando, também espera para dar prioridade
        while (nw > 0 || writer_waiting == true) {
            wait(OKtoread);
        }
        nr = nr + 1;
    }

    procedure release_read() {
        nr = nr - 1;
        // Se este for o último leitor, sinaliza para o próximo escritor
        if (nr == 0) {
            signal(OKtowrite);
        }
    }

    // --- Rotinas do Escritor ---
    procedure request_write() {
        // Sinaliza que um escritor chegou e está esperando
        writer_waiting = true;
        
        // Se houver leitores ou escritores ativos, espera
        while (nr > 0 || nw > 0) {
            wait(OKtowrite);
        }
        nw = nw + 1;
        writer_waiting = false;
    }

    procedure release_write() {
        nw = nw - 1;
        
        // Se houver escritores esperando, dá prioridade a eles
        if (queue(OKtowrite) > 0) {
            signal(OKtowrite);
        } else {
            // Se não, sinaliza para todos os leitores que podem entrar
            signalall(OKtoread);
        }
    }
}

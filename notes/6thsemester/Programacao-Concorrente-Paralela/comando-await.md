É uma abstração presente em toda linguagem de programação, de modo que simplifica a implementação de exclusão mútua e sincronismo.

Seja S um processo/procedimento

<S;> significa que S será executado de modo exclusivo
<await (B);> significa que o próximo comando (que vem após o await) será executado apenas quando B for verdadeiro.
<await (B) S;> Significa que S será executado de modo exclusivo e próximo comando só sera executado quando B for verdadeiro.


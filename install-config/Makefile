MDs=$(wildcard *.md)
FIGs=$(wildcard *.fig)
PDFs=$(MDs:.md=.pdf)
HTMLs=$(MDs:.md=.html)
PNGs=$(FIGs:.fig=.png)
SVGs=$(FIGs:.fig=.svg)

all: $(HTMLs) $(PDFs) $(PNGs) $(SVGs)

%.html: %.md
	pandoc -s -o $@ $<
%.pdf: %.md Makefile $(PNGs) $(SVGs)
	pandoc -V papersize=A4 \
		-V margin-left=2cm \
		-V margin-right=2cm \
		-V margin-top=2cm \
		-V margin-bottom=2cm \
		-o $@ $<

%.png: %.fig
	fig2dev -L png -m 1.5 $< $@

%.svg: %.fig Makefile
	fig2dev -L svg -m 1.2 $< $@

send: tpnote.html
	scp $< labri:public_html/install-config/

MDs=$(wildcard *.md)
FIGs=$(wildcard *.fig)
PDFs=$(MDs:.md=.pdf)
HTMLs=$(MDs:.md=.html)
PNGs=$(FIGs:.fig=.png)

all: $(HTMLs) $(PDFs) $(PNGs)

%.html: %.md
	pandoc -o $@ $<
%.pdf: %.md Makefile
	pandoc -V papersize=A4 \
		-V margin-left=2cm \
		-V margin-right=2cm \
		-V margin-top=2cm \
		-V margin-bottom=2cm \
		-o $@ $<

%.png: %.fig
	fig2dev -L png -m 1.5 $< $@

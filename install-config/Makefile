MDs=$(wildcard *.md)
PDFs=$(MDs:.md=.pdf)
HTMLs=$(MDs:.md=.html)

all: $(HTMLs) $(PDFs)

%.html: %.md
	pandoc -o $@ $<
%.pdf: %.md Makefile
	pandoc -V papersize=A4 \
		-V margin-left=2cm \
		-V margin-right=2cm \
		-V margin-top=2cm \
		-V margin-bottom=2cm \
		-o $@ $<

MDs=$(wildcard *.md)
PDFs=$(MDs:.md=.pdf)
HTMLs=$(MDs:.md=.html)

all: $(HTMLs) $(PDFs)

%.html: %.md
	pandoc -o $@ $<
%.pdf: %.md
	pandoc -o $@ $<

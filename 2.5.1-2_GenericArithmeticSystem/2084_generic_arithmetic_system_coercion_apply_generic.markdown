# Question
Exercise 2.84.

# Answer
## Codes
```scheme
; ***************operation-and-type table**********************
(define op-type-table (make-hash-table))
(define (put op-name data-type procdure)
  (hash-table/put! op-type-table (list op-name data-type) procdure))
(define (get op-name data-type)
  (hash-table/get op-type-table (list op-name data-type) #f))
; ***************operation-and-type table(end)*******************

; ******operation-and-type table for coercion*****************
(define op-type-table-coercion (make-hash-table))
(define (put-coercion data-type-1 data-type-2 procdure)
  (hash-table/put! op-type-table-coercion (list data-type-1 data-type-2) procdure))
(define (get-coercion data-type-1 data-type-2)
  (hash-table/get op-type-table-coercion (list data-type-1 data-type-2) #f))
; *****operation-and-type table for coercion(end)***************

; ******table for the height of one type in a tower of types*****************
(define type-height-table (make-hash-table))
(define (put-height data-type height)
  (hash-table/put! type-height-table data-type height))
(define (get-height data-type)
  (hash-table/get type-height-table data-type #f))
; ******table for the height of one type in a tower of types(end)*****************

; *****************apply generic*****************
(define (type-tag datum)
  (cond ((pair? datum) (car datum))
        (else (error "Bad tagged datum -- TYPE-TAG" datum))))
(define (contents datum)
  (cond ((pair? datum) (cdr datum))
        (else (error "Bad tagged datum -- CONTENTS" datum))))

; (define (apply-generic op . args)
;   (let ((type-tags (map type-tag args)))
;     (let ((proc (get op type-tags)))
;       (if proc
;           (apply proc (map contents args))
;           (error
;             "No method for these types -- APPLY-GENERIC"
;             (list op type-tags))))))
(define (apply-generic op . args)
  (let ((type-tags (map type-tag args)))
    (let ((proc (get op type-tags)))
      (if proc
          (apply proc (map contents args))
          (if (= (length args) 2)
              (let ((type1 (car type-tags))
                    (type2 (cadr type-tags))
                    (a1 (car args))
                    (a2 (cadr args)))
                (let ((height1 (get-height type1))
                      (height2 (get-height type2)))
                  (if (and height1 height2)
                      (if (> height1 height2)
                          (apply-generic op a1 (raise-n (- height1 height2) a2))
                          (apply-generic op (raise-n (- height2 height1) a1) a2))
                      (error "No method for these types"
                                    (list op type-tags)))))
              (error "No method for these types"
                     (list op type-tags)))))))
; *****************apply generic(end)*****************

; *************** data coercion *****************
;(define (scheme-number->complex n)
;  (make-complex-from-real-imag (contents n) 0))
;(put-coercion 'scheme-number 'complex scheme-number->complex)

;(define (rational->scheme-number r)
;  (make-scheme-number (exact->inexact (/ (numer r) (denom r)))))
;(put-coercion 'rational 'scheme-number rational->scheme-number)
; *************** data coercion(end) *****************

; **********************attech type tag********************
(define (attach-tag tag-name contents)
  (cons tag-name contents))
; **********************attech type tag(end)********************

; **********************rectangular package**********************
(define (install-rectangular-package)
  ; internal procedures
  ; 因为scheme中自带real-part等方法,所以自己定义的方法在后面加上字符以区别
  (define (real-part-x z) (car z))
  (define (imag-part-x z) (cdr z))
  (define (make-from-real-imag x y) (cons x y))
  (define (magnitude-x z)
    (sqrt (+ (square (real-part-x z))
             (square (imag-part-x z)))))
  (define (angle-x z)
    (atan (imag-part-x z) (real-part-x z)))
  (define (make-from-mag-ang r a) 
    (cons (* r (cos a)) (* r (sin a))))
  ;; interface to the rest of the system
  (define (tag x) (attach-tag 'rectangular x))
  (put 'real-part-x '(rectangular) real-part-x)
  (put 'imag-part-x '(rectangular) imag-part-x)
  (put 'magnitude-x '(rectangular) magnitude-x)
  (put 'angle-x '(rectangular) angle-x)
  (put 'make-from-real-imag 'rectangular 
       (lambda (x y) (tag (make-from-real-imag x y))))
  (put 'make-from-mag-ang 'rectangular 
       (lambda (r a) (tag (make-from-mag-ang r a))))
  'rectangular-done)
; *******************rectangular package(end)**********************

; **********************polar package***************************
(define (install-polar-package)
  ;; internal procedures
  (define (magnitude-x z) (car z))
  (define (angle-x z) (cdr z))
  (define (make-from-mag-ang r a) (cons r a))
  (define (real-part-x z)
    (* (magnitude-x z) (cos (angle-x z))))
  (define (imag-part-x z)
    (* (magnitude-x z) (sin (angle-x z))))
  (define (make-from-real-imag x y) 
    (cons (sqrt (+ (square x) (square y)))
          (atan y x)))
  ;; interface to the rest of the system
  (define (tag x) (attach-tag 'polar x))
  (put 'real-part-x '(polar) real-part-x)
  (put 'imag-part-x '(polar) imag-part-x)
  (put 'magnitude-x '(polar) magnitude-x)
  (put 'angle-x '(polar) angle-x)
  (put 'make-from-real-imag 'polar
       (lambda (x y) (tag (make-from-real-imag x y))))
  (put 'make-from-mag-ang 'polar 
       (lambda (r a) (tag (make-from-mag-ang r a))))
  'polar-done)
; *******************polar package(end)***************************

; ******************complex package********************
(define (install-complex-package)
  ;; imported procedures from rectangular and polar packages
  (define (make-from-real-imag x y)
    ((get 'make-from-real-imag 'rectangular) x y))
  (define (make-from-mag-ang r a)
    ((get 'make-from-mag-ang 'polar) r a))
  
  ; NOTIC!, add-complex need the methods below
  (define (real-part-x c) (apply-generic 'real-part-x c))
  (define (imag-part-x c) (apply-generic 'imag-part-x c))
  (define (magnitude-x c) (apply-generic 'magnitude-x c))
  (define (angle-x c) (apply-generic 'angle-x c))

  ;; internal procedures
  (define (add-complex z1 z2)
    (make-from-real-imag (+ (real-part-x z1) (real-part-x z2))
                         (+ (imag-part-x z1) (imag-part-x z2))))
  (define (sub-complex z1 z2)
    (make-from-real-imag (- (real-part-x z1) (real-part-x z2))
                         (- (imag-part-x z1) (imag-part-x z2))))
  (define (mul-complex z1 z2)
    (make-from-mag-ang (* (magnitude-x z1) (magnitude-x z2))
                       (+ (angle-x z1) (angle-x z2))))
  (define (div-complex z1 z2)
    (make-from-mag-ang (/ (magnitude-x z1) (magnitude-x z2))
                       (- (angle-x z1) (angle-x z2))))
  (define (equ?-complex z1 z2)
    (and (= (real-part-x z1) (real-part-x z2))
         (= (imag-part-x z1) (imag-part-x z2))))
  (define (=zero?-complex z)
    (and (= (real-part-x z) 0)
         (= (imag-part-x z) 0)))

  ;; interface to rest of the system
  (define (tag z) (attach-tag 'complex z))
  (put 'real-part-x '(complex) real-part-x)
  (put 'imag-part-x '(complex) imag-part-x)
  (put 'magnitude-x '(complex) magnitude-x)
  (put 'angle-x '(complex) angle-x)
  (put 'add '(complex complex)
       (lambda (z1 z2) (tag (add-complex z1 z2))))
  (put 'sub '(complex complex)
       (lambda (z1 z2) (tag (sub-complex z1 z2))))
  (put 'mul '(complex complex)
       (lambda (z1 z2) (tag (mul-complex z1 z2))))
  (put 'div '(complex complex)
       (lambda (z1 z2) (tag (div-complex z1 z2))))
  (put 'equ? '(complex complex) equ?-complex)
  (put '=zero? '(complex) =zero?-complex)

  (put 'make-from-real-imag 'complex
       (lambda (x y) (tag (make-from-real-imag x y))))
  (put 'make-from-mag-ang 'complex
       (lambda (r a) (tag (make-from-mag-ang r a))))
  'complex-done)
; ******************complex package(end)********************

; ******************real package********************
(define (install-real-package)
  (define (tag x)
    (attach-tag 'real x))    
  (put 'add '(real real)
       (lambda (x y) (tag (+ x y))))
  (put 'sub '(real real)
       (lambda (x y) (tag (- x y))))
  (put 'mul '(real real)
       (lambda (x y) (tag (* x y))))
  (put 'div '(real real)
       (lambda (x y) (tag (/ x y))))
  (put 'equ? '(real real) =)
  (put '=zero? '(real)
       (lambda (x) (= x 0)))
  (put 'make 'real
       (lambda (x) (tag x)))
  'real-done)
; ******************real package(end)*****************

; ****************retional number package***************
(define (install-rational-package)
  ;; internal procedures
  (define (numer x) (car x))
  (define (denom x) (cdr x))
  (define (make-rat n d)
    (let ((g (gcd n d)))
      (cons (/ n g) (/ d g))))
  (define (add-rat x y)
    (make-rat (+ (* (numer x) (denom y))
                 (* (numer y) (denom x)))
              (* (denom x) (denom y))))
  (define (sub-rat x y)
    (make-rat (- (* (numer x) (denom y))
                 (* (numer y) (denom x)))
              (* (denom x) (denom y))))
  (define (mul-rat x y)
    (make-rat (* (numer x) (numer y))
              (* (denom x) (denom y))))
  (define (div-rat x y)
    (make-rat (* (numer x) (denom y))
              (* (denom x) (numer y))))
  (define (equ?-rat x y)
    (and (= (numer x) (numer y))
         (= (denom x) (denom y))))
  (define (=zero?-rat x)
    (= (numer x) 0))
  ;; interface to rest of the system
  (define (tag x) (attach-tag 'rational x))
  (put 'add '(rational rational)
       (lambda (x y) (tag (add-rat x y))))
  (put 'sub '(rational rational)
       (lambda (x y) (tag (sub-rat x y))))
  (put 'mul '(rational rational)
       (lambda (x y) (tag (mul-rat x y))))
  (put 'div '(rational rational)
       (lambda (x y) (tag (div-rat x y))))
  (put 'equ? '(rational rational) equ?-rat)
  (put '=zero? '(rational) =zero?-rat)
  (put 'numer '(rational) numer)
  (put 'denom '(rational) denom)

  (put 'make 'rational
       (lambda (n d) (tag (make-rat n d))))
  'rational-done)
; ****************retional number package(end)***************

; ******************integer package********************
(define (install-integer-package)
  (define (tag x)
    (attach-tag 'integer x))    
  (put 'add '(integer integer)
       (lambda (x y) (tag (+ x y))))
  (put 'sub '(integer integer)
       (lambda (x y) (tag (- x y))))
  (put 'mul '(integer integer)
       (lambda (x y) (tag (* x y))))
  (put 'div '(integer integer)
       (lambda (x y) (tag (/ x y))))
  (put 'equ? '(integer integer) =)
  (put '=zero? '(integer)
       (lambda (x) (= x 0)))
  (put 'make 'integer
       (lambda (x) (tag x)))
  'integer-done)
; ******************integer package(end)*****************

; *************set generic arithmetic interface**************
; install packages
(install-rectangular-package)
(install-polar-package)
(install-complex-package)
(install-real-package)
(install-rational-package)
(install-integer-package)

; define number constructors
(define (make-complex-from-real-imag x y)
  ((get 'make-from-real-imag 'complex) x y))
(define (make-complex-from-mag-ang r a)
  ((get 'make-from-mag-ang 'complex) r a))
(define (make-real n)
  ((get 'make 'real) n))
(define (make-rational n d)
  ((get 'make 'rational) n d))
(define (make-integer n)
  ((get 'make 'integer) n))

; define number selectors
(define (real-part-x c) (apply-generic 'real-part-x c))
(define (imag-part-x c) (apply-generic 'imag-part-x c))
(define (magnitude-x c) (apply-generic 'magnitude-x c))
(define (angle-x c) (apply-generic 'angle-x c))
(define (numer r) (apply-generic 'numer r))
(define (denom r) (apply-generic 'denom r))

; define arithmetic operations
(define (add x y) (apply-generic 'add x y))
(define (sub x y) (apply-generic 'sub x y))
(define (mul x y) (apply-generic 'mul x y))
(define (div x y) (apply-generic 'div x y))
(define (equ? x y) (apply-generic 'equ? x y))
(define (=zero? x) (apply-generic '=zero? x))
; *************set generic arithmetic interface(end)**************

; *************raise******************
(put 'raise 'integer
     (lambda (i) (make-rational (contents i) 1)))
(put 'raise 'rational
     (lambda (ra) (make-real (exact->inexact (/ (numer ra) (denom ra))))))
(put 'raise 'real
     (lambda (re) (make-complex-from-real-imag (contents re) 0)))
(define (raise x)
  ((get 'raise (type-tag x)) x))
; *************raise(end)******************

; *************height of type in a tower of types****************
(put-height 'integer 1)
(put-height 'rational 2)
(put-height 'real 3)
(put-height 'complex 4)
; *************height of type in a tower of types(end)****************

; *************raise height of difference n******************
(define (raise-n n x)
  (if (> n 0)
      (raise-n (- n 1) (raise x))
      x))
; *************raise height of difference n(end)******************

; *************debug********************
;(trace-both magnitude-x)
;(trace-both apply-generic)
;(trace-both add)
;(trace-both get-coercion)
(trace-both raise-n)
; *************debug(end)*******************

; ************* test ********************
(define c1 (make-complex-from-real-imag 1 2))
(define c2 (make-complex-from-real-imag 2 3))
(define re1 (make-real 4.5))
(define re2 (make-real 5.6))
(define r1 (make-rational 1 2))
(define r2 (make-rational 8 9))
(define i1 (make-integer 4))
(define i2 (make-integer 5))

;(define r->s (rational->scheme-number r1))
;(define s->c (scheme-number->complex s1))
;(define r+s (add r1 s1))
;(define s+c (add s1 c1))
; ************* test(end) ****************
```

## Running
```
1 ]=> (load "2084_generic_arithmetic_system_coercion_apply_generic.scm")

;Loading "2084_generic_arithmetic_system_coercion_apply_generic.scm"... done
;Value: i2

1 ]=> i1 

;Value 15: (integer . 4)

1 ]=> c1

;Value 16: (complex rectangular 1 . 2)

1 ]=> (get-height 'integer)

;Value: 1

1 ]=> (get-height 'complex)

;Value: 4

1 ]=> (add i1 c1)

[Entering #[compound-procedure 17 raise-n]
    Args: 3
          (integer . 4)]
[Entering #[compound-procedure 17 raise-n]
    Args: 2
          (rational 4 . 1)]
[Entering #[compound-procedure 17 raise-n]
    Args: 1
          (real . 4.)]
[Entering #[compound-procedure 17 raise-n]
    Args: 0
          (complex rectangular 4. . 0)]
[(complex rectangular 4. . 0)
      <== #[compound-procedure 17 raise-n]
    Args: 0
          (complex rectangular 4. . 0)]
[(complex rectangular 4. . 0)
      <== #[compound-procedure 17 raise-n]
    Args: 1
          (real . 4.)]
[(complex rectangular 4. . 0)
      <== #[compound-procedure 17 raise-n]
    Args: 2
          (rational 4 . 1)]
[(complex rectangular 4. . 0)
      <== #[compound-procedure 17 raise-n]
    Args: 3
          (integer . 4)]
;Value 18: (complex rectangular 5. . 2)
```

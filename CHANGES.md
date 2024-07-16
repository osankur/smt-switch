# Changes in the current fork
This is a fork from the commit 445b5bc5172cc4a56db121e5ba4c7a5e14147bd5 used in Pono (as of March 28, 2024).
These changed were done in this commit which is not up to date because the currently available version of Pono relies on this.

## Quantifier elimination
The AbsSmtSolver interface was extended with support for quantifier elimination.
This is currently implemented only for cvc5 (at least msat and z3 also support this).

## Bug Fix: TermTranslator::infixize_rational
This function was unable to parse rationals with negative nominator or denominators such as "(/ (- 3) 5)".
Because not all solvers have the unary minus operator (e.g. msat), a new function was added: `TermTranslator::parse_rational`.
This parses nominator and the denominator as nonnegative numbers and returns whether the value must be negated.

## Bug Fix: TermTranslator::value_from_smt2
This function could not parse negative rational values such as "(/ (- 3) 5)" due to 
`Term Cvc5Solver::make_term(std::string val, const Sort & sort, uint64_t base) const` not being able to handle negative values.
The latter function was modified to bring support for parsing negative integers.

## Bug Workaround: TermTranslator::transfer_term 
This function always introduced bitvector terms even if the given formula does not have these.
This was due to the following part:

```
        // if (!check_sortedness(op, cached_children))
        // {
        //   /* NOTE: interesting behavior here
        //      if transferring between two solvers that alias sorts
        //      e.g. two different instances of BTOR
        //      the sorted-ness check will still fail for something like
        //      Ite(BV{1}, BV{8}, BV{8})
        //      so we'll reach this point and cast
        //      but the cast won't actually do anything for BTOR
        //      in other words, check_sortedness is not guaranteed
        //      to hold after casting */
        //   cache[t] = cast_op(op, cached_children);
```

This part is commented, which means that the case described in these comments might fail.

Some tests in test-term-translator involving bv theories fail currently due to the above modification.
The present version of the library must not be used with bv theories.

However, for formulas purely in LRA or LIA, this no longer introduces bitvectors.

The following part was also modified:
```it = value_from_smt2(
   it->print_value_as(it->get_sort()->get_sort_kind()), it->get_sort());
```
in order to explicitly build negated terms which behave differently among solvers.

## Bug Fix: Const_Rational in CVC5
In CVC5 rational values are under unary Const_Rational operator unlike in other solvers such sa MSAT.
Currently, get_op() throws an exception since this operator does not have an abstract counterpart.
As a result, creating a rational value term in CVC5 and transferring it back to another CVC5 solver fails with an exception.

In the function `Op Cvc5Term::get_op() const`, I added:

  ```
  // special cases
  if (cvc5_kind == cvc5::Kind::CONST_ARRAY || cvc5_kind == cvc5::Kind::CONST_RATIONAL)
  {
    // constant array and const rationals are values in smt-switch
    return Op();
  }
  ```
This issue was already handled in this way for const array.

## Bug Fix: MSAT and CVC5 not being able to make terms out of negative numbers
`MsatSolver::make_term` and `Term Cvc5Solver::make_term(std::string val, const Sort & sort, uint64_t base) const` were updated so that a given string "(- 5)" can be parsed as the corresponding term built. 


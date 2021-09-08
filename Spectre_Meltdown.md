## Cause - speculative execution
In order to make CPUs/chips faster, chip developers added a function to processors called "Speculative Execution", which allows 
the computer to guess what user might do next and perform necessary calculations for those possible outcomes in advance.

If the prediction is false, the calculations are useless and then get trashed. The trashed calculations - are not really protected.

## Spectre and Meltdown
Spectre and Meltdown exploit this feature by using malicious code that tricks a computer into "speculatively loading" information
it wouldn't normally have access to.

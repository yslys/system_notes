## CPP STL cheat sheet
+ How to convert an int to a string?
  + `string s = to_string(x)`
+ How to check if a `key` exists in `unordered_map<> m`?
  + `if (unordered_map.count(key) == 0)' then key doesn't exist
  + `unordered_map.count(key)` can return at most 1 due to it is a dictionary.
+ How to sort a container, say `string`?
  + `sort(s.begin(), s.end())`
+ What to do first when solving linked list problems?
  + Use a dummy node. At the end, return `dummy->next`
+ Given an unsorted array, how to remove duplicates and sort?
  + A. insert each element into a set, since set is ordered.
    + `for (int &num : arr) s.insert(num);`. The reason we use `&` is because we do not want to make a copy of `num` in `arr`. But be careful, if we use reference, any changes to `num` will be reflected in `arr`.
  + B. `sort(array)` first, then `array.erase()` to remove duplicates
    + Before doing the above, we should always create a copy of the data:
      + `std::vector<int> sortedUniqueNumbers = arr;`
    + Sort first, due to the behavior of erase
      + `std::sort(sortedUniqueNumbers.begin(), sortedUniqueNumbers.end());`
    + Then erase:
      + `sortedUniqueNumbers.erase(std::unique(sortedUniqueNumbers.begin(), sortedUniqueNumbers.end()), sortedUniqueNumbers.end());`
        + `vector.erase(v.begin(), v.end())`: this function takes in two parameters that are iterators.
      + `std::unique(vector)`: this will remove all the extra consecutive numbers in the vector.
        + e.g.  `{ 1, 1, 3, 3, 3, 10, 1, 3, 3, 7, 7, 8 }` -> `{ 1, 3, 10, 1, 3, 7, 8 }`
        + This is why we need to sort the vector first, then call `unique()` function
      + Return value of `std::unique(vector)`: an iterator pointing to the end of the de-duplicated array, and we treat that iterator as the begin() of the index that we want to erase.      
+ How to obtain a substring of a `string s`?
  + Use `s.substr(start_idx, length)`.

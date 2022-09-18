+++
title = "Letter Combinations of a Phone Number"
date = 2022-09-09
draft = false

[taxonomies]
categories = ["Coding Problems"]
tags = ["leetcode","c++"]

[extra]
lang = "en"
toc = true
show_comment = true
math = false
mermaid = true
cc_license = true
outdate_warn = true
outdate_warn_days = 120
+++

## Link
- [https://leetcode.com/problems/letter-combinations-of-a-phone-number/](https://leetcode.com/problems/letter-combinations-of-a-phone-number/)

## Rationale
To solve this problem is useful to use backtracking and keep the state of the current letters in a string.
Backtrack acts as follows:
- Check if the final condition is met and add the computed solution in the the vector of strings,
- Create a new candidate solution by pushing back a new element,
- Perform backtrack by calling recursively the backtrack function,
- Remove the last element from the string

## Solution

```cpp
class Solution {
public:
    
    unordered_map<char, std::string> letter_map = 
    {
        {'2', "abc"},
        {'3', "def"},
        {'4', "ghi"},
        {'5', "jkl"},
        {'6', "mno"},
        {'7', "pqrs"},
        {'8', "tuv"},
        {'9', "wxyz"}
    };
    
    vector<string> total;
    
    void dfs(const string& digits, int i, char curr, string& res)
    {
        if(i >= digits.size())
        {
            total.push_back(res);
            return;
        }
        
        for(auto let : letter_map[digits[i]])
        {
            res.push_back(let);
            dfs(digits, i + 1, let, res);
            res.pop_back();
        }
    }
    
    vector<string> letterCombinations(string digits) {
        if(digits.empty())
        {
            return {};
        }
        
        string res = "";
        dfs(digits, 0, 'i', res);
        return total;
    }
};

```
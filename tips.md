1. UILabel preferredMaxLayoutWidth 需要设置，特定场景下会有布局问题
2. UIStackView semanticContentAttribute = UIView.appearance().semanticContentAttribute， 如果不设置的话，在某些RTL语言下会出现显示错误

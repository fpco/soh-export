# IDEs and Linters

<!DOCTYPE html>
<p>Incomplete. Need to add:</p><ul><li>Emacs:
<i> haskell-mode + hlint
</i> stylish-haskell
<i> Additional:
  </i> multi-tags search
  * projectile for projects</li><li>Vim - expand, show simple usage of each without redoing the libraries' own documentation</li><li>other IDEs not listed here</li></ul><h2>Emacs</h2><p>Emacs is one of most popular Haskell editors out there. If you are
able to use it, you can get a very nice development environment.</p><h3>Prerequisites</h3><h4>Cask (or use-package)</h4><p>It's assumed in this tutorial you have modern package manager Cask installed.
You can install it by following the
<a href="http://cask.readthedocs.org/en/latest/index.html">cask user guide</a>. For
older Emacs versions (&lt; 24), please figure out other method of
package-installation.</p><p>Make sure a <code>Cask</code> file in your <code>.emacs.d</code> directory has marmelade
repo, ensure it looks something like this:</p><pre><code class="elisp">(source gnu)
(source melpa)
(source marmalade)</code></pre><p>And check that your <code>~/.emacs</code> file has:</p><pre><code class="elisp">(require 'cask "~/.cask/cask.el")
(cask-initialize)</code></pre><p>After adding dependencies to your <code>Cask</code> file, run <code>cask install</code> in
that directory.</p><p>If instead of <code>Cask</code> you are using <a href="https://github.com/jwiegley/use-package">use-package</a>
to manage your packages, then make sure that you have this in your <code>init.el</code> file:</p><pre><code class="elisp">(setq package-archives
      '(("gnu"         . "http://elpa.gnu.org/packages/")
        ("marmalade"   . "http://marmalade-repo.org/packages/")
        ("melpa" . "http://melpa.milkbox.net/packages/")))
(package-initialize)
(require 'use-package)</code></pre><p>Make sure that you install <code>use-package</code> from the package manager. Following tutorial
should be straightforward.</p><h4>exec-path-from-shell</h4><p>On MacOS X you will also need
<a href="https://github.com/purcell/exec-path-from-shell">exec-path-from-shell</a>
in order to get your environment variables from your shell (bash or
zsh) into your emacs, so that it inherits your <code>$PATH</code> and sees proper
<code>~/.cabal/bin/cabal</code>, for example. To install, add following to your
<code>Cask</code> file:</p><pre><code class="elisp">(depends-on "exec-path-from-shell")</code></pre><p>and run <code>cask install</code> from <code>Cask</code>-file directory. Add following to your <code>~/.emacs</code>:</p><pre><code class="elisp">(when (memq window-system '(mac ns))
  (exec-path-from-shell-initialize))</code></pre><h3>haskell-mode</h3><p>First of all, you should use
<a href="https://github.com/haskell/haskell-mode">haskell-mode</a> for editing
<code>.hs</code> and <code>.cabal</code> files. Add this line to your <code>Cask</code>:</p><pre><code class="elisp">(depends-on "haskell-mode")</code></pre><p>and run <code>cask install</code> in that file's directory. This should install a
fresh <code>haskell-mode</code> for you. Restart your emacs and you should be
ready to open any <code>.cabal</code> or <code>.hs</code> file for editing. You will see
syntax highlighted, you'll be able to run commands like <code>M-x
haskell-process-load-file</code>.</p><p>However, you can't yet use <code>haskell-mode</code> in full because you haven't
configured indentation mode and interaction. Go read
<a href="https://github.com/haskell/haskell-mode/wiki/Indentation">haskell-mode tutorial on setting the indentation-mode you like</a>
and
<a href="https://github.com/haskell/haskell-mode/wiki/Haskell-Interactive-Mode-Setup">Haskell Interactive Mode Setup</a>
pages, or just put something like this to your <code>.emacs</code> (if you
already have <code>custom-set-variables</code> in your <code>.emacs</code> -- extend it):</p><pre><code class="elisp">(defun turn-on-subword-mode ()
  (interactive)
  (subword-mode 1))
(defun my-haskell-mode-hook ()
  (define-key haskell-mode-map (kbd "C-c C-l") 'haskell-process-load-or-reload)
  (define-key haskell-mode-map (kbd "C-`") 'haskell-interactive-bring)
  (define-key haskell-mode-map (kbd "C-c C-t") 'haskell-process-do-type)
  (define-key haskell-mode-map (kbd "C-c C-i") 'haskell-process-do-info)
  (define-key haskell-mode-map (kbd "C-c C-c") 'haskell-process-cabal-build)
  (define-key haskell-mode-map (kbd "C-c C-k") 'haskell-interactive-mode-clear)
  (define-key haskell-mode-map (kbd "C-c c") 'haskell-process-cabal)
  (define-key haskell-mode-map (kbd "SPC") 'haskell-mode-contextual-space))
(custom-set-variables
 '(haskell-mode-hook
   (quote
    (interactive-haskell-mode 
     turn-on-haskell-indent 
     turn-on-subword-mode
     turn-on-haskell-decl-scan 
     my-haskell-mode-hook)))
 '(haskell-process-type 'cabal-repl))</code></pre><p>Now you're more or less ready, but this is just a beginning. Go ahead,
open some <code>.hs</code> file and press <code>C-h ? b</code>. You will see list of
bindings available. Search for <code>haskell-</code> to see all haskell-related
bindings. Also, run <code>M-x haskell-</code> and press <code>TAB</code> for list of
available commands starting with <code>haskell-</code>. There's plenty of them!</p><p>Press <code>C-c C-l</code> to open a <code>cabal repl</code> shell with current module
loaded. You can easily play with your code like this. Also, pressing
<code>C-c C-i</code> will inspect symbol under cursor and show you some info. Neat!</p><h3>ghc-mod</h3><p><a href="http://www.mew.org/~kazu/proj/ghc-mod/en/">Ghc-mod</a> is a tool that
helps your editor become more close to an IDE. It lets you do things
like blazingly-fast error-reporting upon save, autocompletion and more.</p><p>First, install ghc-mod itself:</p><pre><code>cabal install ghc-mod</code></pre><p>Then, add this to your <code>Cask</code> file:</p><pre><code class="elisp">(depends-on "ghc")</code></pre><p>And this to your <code>~/.emacs</code>:</p><pre><code>(autoload 'ghc-init "ghc" nil t)
(autoload 'ghc-debug "ghc" nil t)</code></pre><p>and extend your <code>my-haskell-mode-hook</code> with call to <code>(ghc-init)</code> like
this:</p><pre><code class="elisp">(defun my-haskell-mode-hook ()
  (ghc-init)
  ; ... all previous contents
  )</code></pre><p>After this is done, relaunch your Emacs (or re-eval config)
and open some <code>.hs</code>-file. Make a mistake and save the file, mistake
should now become underlined with red, and pressing <code>M-S-?</code> while
keeping curstor on top of it will show the error, and <code>M-n</code> <code>M-p</code>
should navigate between next/prev errors.</p><p>Check out
<a href="http://www.mew.org/~kazu/proj/ghc-mod/en/">ghc-mod homepage</a> and
other resources to know more.</p><h3>TAGS</h3><p>In order to get a "goto by name" functionality you can use standard
<a href="https://www.gnu.org/software/emacs/manual/html_node/emacs/Tags.html">Emacs TAGS support</a>. Haskell
has a special program, called <code>hasktags</code>:</p><pre><code>cabal install hasktags</code></pre><p>In order to generate tags, you can use <code>haskell-mode</code>'s <code>M-x
haskell-process-generate-tags</code>, or you can manually run <code>hasktags -e.</code>
in your project's root.</p><p><code>haskell-mode</code> replaces standard <code>M-.</code> tag-search with it's own,
trying to search via ghci first, and only then via standard TAGS
mechanism.</p><h3>Auto-completion via company-ghc</h3><p>Auto-completion via <a href="https://github.com/iquiw/company-ghc">company-ghc</a> is pretty simple. Put this to your <code>Cask</code>:</p><pre><code class="elisp">(depends-on "company")
(depends-on "company-ghc")</code></pre><p>Then put this to your <code>.emacs</code>:</p><pre><code class="elisp">(require 'company)
(add-hook 'after-init-hook 'global-company-mode)
(add-to-list 'company-backends 'company-ghc)</code></pre><p>Voila! Beautiful auto-complete from <code>ghc-mod</code> works like a charm.</p><h2>Vim</h2><p>Suggested plugins</p><ul><li><a href="https://github.com/Shougo/vimproc.vim">Vimproc</a> (required when installing ghcmod-vim)</li><li><a href="https://github.com/eagletmt/ghcmod-vim">ghcmod-vim</a> - a wrapper for the multifaceted <a href="https://github.com/kazu-yamamoto/ghc-mod">ghc-mod</a> command-line tool</li><li><a href="https://github.com/dag/vim2hs">vim2hs</a> - A configurable collection of syntax highlighting and keyword shortcuts tailored for Haskell development</li><li><a href="https://github.com/scrooloose/syntastic">Syntastic</a> - syntax checking and linting</li><li><a href="https://github.com/eagletmt/neco-ghc">neco-ghc</a> - Haskell/GHC completion plugin</li><li><a href="https://github.com/Shougo/neocomplete.vim">neocomplete</a> - General-purpose keyword completion engine, fed by neco-ghc</li><li><a href="https://github.com/timmytofu/vim-cabal-context">Cabal Context</a> - Sets pwd to the directory of the nearest cabal file, useful for bouncing between different libraries</li></ul>
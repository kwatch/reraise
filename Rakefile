# -*- coding: utf-8 -*-


files = %w[
  bin/reraise
  bin/auto-reraise
  README.md
]

def edit_file(filename)
  File.open(filename, 'r+', encoding: 'utf-8') do |f|
    s1 = f.read()
    s2 = yield s1
    if s1 != s2
      f.rewind()
      f.truncate(0)
      f.write(s2)
      true
    else
      false
    end
  end
end

desc "edit files (requires 'rel=X.X.X')"
task :edit do
  rel = ENV['rel']
  rel && !rel.empty?  or
    raise "requires 'rel=X.X.X'"
  rel =~ /\A\d+\.\d+\.\d+\z/  or
    raise "'rel=#{rel}': invalid format."
  files.each do |filename|
    changed = edit_file(filename) {|s|
      s = s.gsub(/\$Release(?::.*?)?\$/, "$Release: #{rel} $")
      s
    }
    if changed
      puts "[U] #{filename}"
    else
      puts "[-] #{filename}"
    end
  end
end

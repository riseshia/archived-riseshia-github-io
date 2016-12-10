require 'time'

desc 'create a new draft post'
task :post do
    title = ENV['title']
    time = Time.now
    slug = "#{time.strftime('%Y-%m-%d')}-#{title.downcase.gsub(/[^\w]+/, '-')}"

    file = File.join(
        File.dirname(__FILE__),
        '_posts',
        slug + '.md'
    )

    File.open(file, "w") do |f|
        f << <<-EOS.gsub(/^        /, '')
        ---
        layout: post
        title: "#{title}"
        date: #{time.strftime('%Y-%m-%d %T %z')}
        categories:
        ---

        EOS
    end

    system ("#{ENV['EDITOR']} #{file}")
end

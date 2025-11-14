classdef image123 < matlab.apps.AppBase

    % Properties that correspond to app components
    properties (Access = public)
        UIFigure                  matlab.ui.Figure
        FILTERTYPEDropDown        matlab.ui.control.DropDown
        FILTERTYPEDropDownLabel   matlab.ui.control.Label
        IMAGEPROCESSINGANDEDGEDETECTIONLabel  matlab.ui.control.Label
        THRESHOLDSlider           matlab.ui.control.Slider
        THRESHOLDSliderLabel      matlab.ui.control.Label
        KERNELSIZEDropDown        matlab.ui.control.DropDown
        KERNELSIZEDropDownLabel   matlab.ui.control.Label
        EDGEDETECTIONDropDown     matlab.ui.control.DropDown
        Label                     matlab.ui.control.Label
        SAVEIMAGEButton           matlab.ui.control.Button
        RESETButton               matlab.ui.control.Button
        APPLYEDGEDETECTIONButton  matlab.ui.control.Button
        APPLYFILTERButton         matlab.ui.control.Button
        LOADIMAGEButton           matlab.ui.control.Button
        UIAxes2                   matlab.ui.control.UIAxes
        UIAxes                    matlab.ui.control.UIAxes
    end

    
    properties (Access = private)
        OriginalImage  % Original loaded image
        ProcessedImage % Processed image
        CurrentImage   % Current working image
        % Description
    end
    
    methods (Access = private)
        % ---------- FILTER KERNEL --------------------
        function kernel = getFilterKernel(app, filterType, kernelSize)
            switch filterType

                case 'Average (Low-Pass)'
                    kernel = ones(kernelSize) / (kernelSize^2);

                case 'Gaussian'
                    sigma = kernelSize/6;
                    kernel = fspecial('gaussian',[kernelSize kernelSize], sigma);

                case 'Sharpening'
                    if kernelSize == 3
                        kernel=[0 -1 0; -1 5 -1; 0 -1 0];
                    else
                        kernel=[-1 -1 -1; -1 9 -1; -1 -1 -1];
                    end

                case 'Laplacian'
                    kernel=fspecial('laplacian',0.2);

                case 'Motion Blur'
                    kernel = fspecial('motion',kernelSize,45);

                case 'Unsharp Mask'
                    kernel = fspecial('unsharp');

                case 'High-Pass'
                    kernel = [-1 -1 -1; -1 8 -1; -1 -1 -1];
            end
        end

        % ---------- APPLY CONVOLUTION --------------------
        function filtered = applyConvolution(app, img, kernel)
            if size(img,3)==3
                filtered=zeros(size(img));
                for i=1:3
                    filtered(:,:,i)=conv2(double(img(:,:,i)),kernel,'same');
                end
            else
                filtered = conv2(double(img),kernel,'same');
            end
            filtered = uint8(filtered);
        end

        % ---------- CUSTOM EDGE DETECTION --------------------
        function edges = detectEdgesConvolution(app, img, method, threshold)
            if size(img,3)==3
                grayImg = rgb2gray(img);
            else
                grayImg = img;
            end

            grayImg = double(grayImg);

            switch method
                case 'Sobel'
                    Gx=[-1 0 1; -2 0 2; -1 0 1];
                    Gy=[-1 -2 -1; 0 0 0; 1 2 1];

                case 'Prewitt'
                    Gx=[-1 0 1; -1 0 1; -1 0 1];
                    Gy=[-1 -1 -1; 0 0 0; 1 1 1];

                case 'Roberts'
                    Gx=[1 0; 0 -1];
                    Gy=[0 1; -1 0];

                case 'Laplacian'
                    kernel=[0 1 0; 1 -4 1; 0 1 0];
                    edges=abs(conv2(grayImg,kernel,'same'));
                    edges = edges > threshold * max(edges(:));
                    edges=uint8(edges)*255;
                    return;

                case 'Canny'
                    edges = edge(uint8(grayImg),'canny',threshold);
                    edges = uint8(edges)*255;
                    return;

                case 'Laplacian of Gaussian'
                    h = fspecial('gaussian',[5 5],1);
                    smoothed = conv2(grayImg,h,'same');
                    kernel=[0 1 0; 1 -4 1; 0 1 0];
                    edges=abs(conv2(smoothed,kernel,'same'));
                    edges = edges > threshold * max(edges(:));
                    edges=uint8(edges)*255;
                    return;
            end

            Ex = conv2(grayImg,Gx,'same');
            Ey = conv2(grayImg,Gy,'same');
            magnitude = sqrt(Ex.^2 + Ey.^2);

            edges = magnitude > threshold * max(magnitude(:));
            edges = uint8(edges)*255;
        end
    end
   
    

    % Callbacks that handle component events
    methods (Access = private)

        % Button pushed function: LOADIMAGEButton
        function LOADIMAGEButtonPushed(app, event)
        [file,path] = uigetfile({'*.jpg;*.png;*.jpeg'});
            if isequal(file,0), return; end

            app.OriginalImage = imread(fullfile(path,file));
            imshow(app.OriginalImage,'Parent',app.UIAxes);
        end

        % Button pushed function: APPLYFILTERButton
        function APPLYFILTERButtonPushed(app, event)
       if isempty(app.OriginalImage)
                uialert(app.UIFigure,'Load an image first!','Error');
                return;
            end

            img = rgb2gray(app.OriginalImage);
            k = str2double(app.KERNELSIZEDropDown.Value);

            switch app.FILTERTYPEDropDown.Value

                case 'Low Pass'
                    h = fspecial('average',k);
                    result = imfilter(img,h);

                case 'High Pass'
                    h = [-1 -1 -1; -1 8 -1; -1 -1 -1];
                    result = imfilter(img,h);

                case 'Gaussian Blur'
                    sigma = k/3;
                    h = fspecial('gaussian',k,sigma);
                    result = imfilter(img,h);

                case 'Median Filter'
                    result = medfilt2(img,[k k]);

                case 'Sharpen'
                    result = imsharpen(img,'Radius',k,'Amount',1.2);
            end

            app.ProcessedImage = result;
            imshow(result,'Parent',app.UIAxes2);
        
        end

        % Button pushed function: APPLYEDGEDETECTIONButton
        function APPLYEDGEDETECTIONButtonPushed(app, event)
         
    if isempty(app.OriginalImage)
        uialert(app.UIFigure,'Load an image first!','Error');
        return;
    end

    img = rgb2gray(app.OriginalImage);
    t = app.THRESHOLDSlider.Value;

    % Make threshold safe
    if t < 0 || t > 1
        t = 0.1;
    end

    switch app.EDGEDETECTIONDropDown.Value

        case 'Sobel'
            if t > 0.3
                t = 0.1;
            end
            edges = edge(img,'sobel',t);

        case 'Prewitt'
            if t > 0.3
                t = 0.1;
            end
            edges = edge(img,'prewitt',t);

        case 'Roberts'
            if t > 0.3
                t = 0.1;
            end
            edges = edge(img,'roberts',t);

        case 'Canny'
            low = t/2;
            high = t;
            if low <= 0, low = 0.05; end
            if high <= low, high = low + 0.05; end
            edges = edge(img,'canny',[low high]);

        case 'LoG'
            if t > 0.3
                t = 0.1;
            end
            edges = edge(img,'log',t);

    end

    app.ProcessedImage = edges;
    imshow(edges,'Parent',app.UIAxes2);
        end

        % Button pushed function: RESETButton
        function RESETButtonPushed(app, event)
           cla(app.UIAxes);
            cla(app.UIAxes2);
            app.OriginalImage = [];
            app.ProcessedImage = [];
        end

        % Button pushed function: SAVEIMAGEButton
        function SAVEIMAGEButtonPushed(app, event)
           if isempty(app.ProcessedImage)
                uialert(app.UIFigure,'No processed image to save!','Error');
                return;
            end

            [file,path] = uiputfile({'*.png';'*.jpg';'*.bmp'},'Save Image');

            if isequal(file,0)
                return;
            end

            imwrite(app.ProcessedImage, fullfile(path,file));

            uialert(app.UIFigure,'Image Saved Successfully!','Saved');
        end

        % Value changed function: THRESHOLDSlider
        function THRESHOLDSliderValueChanged(app, event)
         val = app.THRESHOLDSlider.Value;
            app.THRESHOLDSliderLabel.Text = sprintf('THRESHOLD: %.2f',val);
        end
    end

    % Component initialization
    methods (Access = private)

        % Create UIFigure and components
        function createComponents(app)

            % Create UIFigure and hide until all components are created
            app.UIFigure = uifigure('Visible', 'off');
            app.UIFigure.Color = [0.7569 0.8784 0.7216];
            app.UIFigure.Position = [100 100 640 480];
            app.UIFigure.Name = 'MATLAB App';

            % Create UIAxes
            app.UIAxes = uiaxes(app.UIFigure);
            title(app.UIAxes, 'ORIGINAL IMAGE')
            zlabel(app.UIAxes, 'Z')
            app.UIAxes.FontWeight = 'bold';
            app.UIAxes.Position = [26 199 300 185];

            % Create UIAxes2
            app.UIAxes2 = uiaxes(app.UIFigure);
            title(app.UIAxes2, 'PROCESSED IMAGE')
            zlabel(app.UIAxes2, 'Z')
            app.UIAxes2.FontWeight = 'bold';
            app.UIAxes2.Position = [325 199 300 185];

            % Create LOADIMAGEButton
            app.LOADIMAGEButton = uibutton(app.UIFigure, 'push');
            app.LOADIMAGEButton.ButtonPushedFcn = createCallbackFcn(app, @LOADIMAGEButtonPushed, true);
            app.LOADIMAGEButton.FontWeight = 'bold';
            app.LOADIMAGEButton.Position = [40 151 100 22];
            app.LOADIMAGEButton.Text = 'LOAD IMAGE';

            % Create APPLYFILTERButton
            app.APPLYFILTERButton = uibutton(app.UIFigure, 'push');
            app.APPLYFILTERButton.ButtonPushedFcn = createCallbackFcn(app, @APPLYFILTERButtonPushed, true);
            app.APPLYFILTERButton.FontWeight = 'bold';
            app.APPLYFILTERButton.Position = [226 151 100 22];
            app.APPLYFILTERButton.Text = 'APPLY FILTER';

            % Create APPLYEDGEDETECTIONButton
            app.APPLYEDGEDETECTIONButton = uibutton(app.UIFigure, 'push');
            app.APPLYEDGEDETECTIONButton.ButtonPushedFcn = createCallbackFcn(app, @APPLYEDGEDETECTIONButtonPushed, true);
            app.APPLYEDGEDETECTIONButton.FontWeight = 'bold';
            app.APPLYEDGEDETECTIONButton.Position = [414 151 163 22];
            app.APPLYEDGEDETECTIONButton.Text = 'APPLY EDGE DETECTION';

            % Create RESETButton
            app.RESETButton = uibutton(app.UIFigure, 'push');
            app.RESETButton.ButtonPushedFcn = createCallbackFcn(app, @RESETButtonPushed, true);
            app.RESETButton.FontWeight = 'bold';
            app.RESETButton.Position = [346 104 100 22];
            app.RESETButton.Text = 'RESET';

            % Create SAVEIMAGEButton
            app.SAVEIMAGEButton = uibutton(app.UIFigure, 'push');
            app.SAVEIMAGEButton.ButtonPushedFcn = createCallbackFcn(app, @SAVEIMAGEButtonPushed, true);
            app.SAVEIMAGEButton.FontWeight = 'bold';
            app.SAVEIMAGEButton.Position = [487 104 100 22];
            app.SAVEIMAGEButton.Text = 'SAVE IMAGE';

            % Create Label
            app.Label = uilabel(app.UIFigure);
            app.Label.HorizontalAlignment = 'right';
            app.Label.FontWeight = 'bold';
            app.Label.Position = [28 64 112 22];
            app.Label.Text = 'EDGE DETECTION';

            % Create EDGEDETECTIONDropDown
            app.EDGEDETECTIONDropDown = uidropdown(app.UIFigure);
            app.EDGEDETECTIONDropDown.Items = {'Sobel', 'Prewitt', 'Roberts', 'Canny', 'LoG'};
            app.EDGEDETECTIONDropDown.FontWeight = 'bold';
            app.EDGEDETECTIONDropDown.Position = [155 64 100 22];
            app.EDGEDETECTIONDropDown.Value = 'Sobel';

            % Create KERNELSIZEDropDownLabel
            app.KERNELSIZEDropDownLabel = uilabel(app.UIFigure);
            app.KERNELSIZEDropDownLabel.HorizontalAlignment = 'right';
            app.KERNELSIZEDropDownLabel.FontWeight = 'bold';
            app.KERNELSIZEDropDownLabel.Position = [28 32 84 22];
            app.KERNELSIZEDropDownLabel.Text = 'KERNEL SIZE';

            % Create KERNELSIZEDropDown
            app.KERNELSIZEDropDown = uidropdown(app.UIFigure);
            app.KERNELSIZEDropDown.Items = {'3', '5', '7', '9', '11'};
            app.KERNELSIZEDropDown.FontWeight = 'bold';
            app.KERNELSIZEDropDown.Position = [127 32 100 22];
            app.KERNELSIZEDropDown.Value = '3';

            % Create THRESHOLDSliderLabel
            app.THRESHOLDSliderLabel = uilabel(app.UIFigure);
            app.THRESHOLDSliderLabel.HorizontalAlignment = 'right';
            app.THRESHOLDSliderLabel.FontWeight = 'bold';
            app.THRESHOLDSliderLabel.Position = [335 53 80 22];
            app.THRESHOLDSliderLabel.Text = 'THRESHOLD';

            % Create THRESHOLDSlider
            app.THRESHOLDSlider = uislider(app.UIFigure);
            app.THRESHOLDSlider.ValueChangedFcn = createCallbackFcn(app, @THRESHOLDSliderValueChanged, true);
            app.THRESHOLDSlider.FontWeight = 'bold';
            app.THRESHOLDSlider.Position = [437 62 150 3];

            % Create IMAGEPROCESSINGANDEDGEDETECTIONLabel
            app.IMAGEPROCESSINGANDEDGEDETECTIONLabel = uilabel(app.UIFigure);
            app.IMAGEPROCESSINGANDEDGEDETECTIONLabel.FontSize = 18;
            app.IMAGEPROCESSINGANDEDGEDETECTIONLabel.FontWeight = 'bold';
            app.IMAGEPROCESSINGANDEDGEDETECTIONLabel.Position = [173 419 404 23];
            app.IMAGEPROCESSINGANDEDGEDETECTIONLabel.Text = 'IMAGE PROCESSING AND EDGE DETECTION';

            % Create FILTERTYPEDropDownLabel
            app.FILTERTYPEDropDownLabel = uilabel(app.UIFigure);
            app.FILTERTYPEDropDownLabel.HorizontalAlignment = 'right';
            app.FILTERTYPEDropDownLabel.FontWeight = 'bold';
            app.FILTERTYPEDropDownLabel.Position = [31 104 81 22];
            app.FILTERTYPEDropDownLabel.Text = 'FILTER TYPE';

            % Create FILTERTYPEDropDown
            app.FILTERTYPEDropDown = uidropdown(app.UIFigure);
            app.FILTERTYPEDropDown.Items = {'Low Pass', 'High Pass', 'Gaussian Blur', 'Median Filter', 'Sharpen'};
            app.FILTERTYPEDropDown.FontWeight = 'bold';
            app.FILTERTYPEDropDown.Position = [127 104 100 22];
            app.FILTERTYPEDropDown.Value = 'Low Pass';

            % Show the figure after all components are created
            app.UIFigure.Visible = 'on';
        end
    end

    % App creation and deletion
    methods (Access = public)

        % Construct app
        function app = image123

            % Create UIFigure and components
            createComponents(app)

            % Register the app with App Designer
            registerApp(app, app.UIFigure)

            if nargout == 0
                clear app
            end
        end

        % Code that executes before app deletion
        function delete(app)

            % Delete UIFigure when app is deleted
            delete(app.UIFigure)
        end
    end
end
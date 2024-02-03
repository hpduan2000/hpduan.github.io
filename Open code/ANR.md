## main.m
    %% Record-Decomposition Incorporation (RDI) main by H.P.Duan; hpduan2000@csu.edu.cn
    %  Ref: Effects of Near-Fault Motions and Artificial Pulse-Type Ground Motions on Super-Span Cable-Stayed Bridge Systems
    %  (2017, Journal of Bridge Engineering, S.Li, et al.)
    clear
    clc
    warning off
    % Read GM records of velocity time series
    filepath = 'D:\HPduan\post_MATLAB\SolveResults_SCI4\ANR';  % floder
    filename1 = 'RSN1045_NORTHR_WPI046.VT2';  % file name volicity
    filename2 = 'RSN1045_NORTHR_WPI046.AT2';  % file name acceleration
    [v_series, dt, ~, ~] = getAmpDtPEER(filepath, filename1);
    [a_series, ~, ~, ~] = getAmpDtPEER(filepath, filename2);
    % Decomposition
    Tp = 8;      % pluse period (Tp)
    alpha_1 = 0.77; % empirical coefficient
    [v_PTR, v_BGR] = FourthButterworth(v_series, dt, Tp, alpha_1);
    % Incorporation
    Ap = 65;    % pulse amplitude
    fp = 0.3;   % mathematical pulse frequency
    t0 = 5.9;   % appear time of mathematical pulse amplitude
    gama = 1.8; % random par
    v_ = 200;   % phase par
    v_APTR = SynthesisPulse(v_series, dt, Ap, fp, t0, gama, v_);
    % pseudo-spectral velocity
    a_PTR = (diff(v_PTR)/dt)*1e-2;   % m/s2
    a_APTR = (diff(v_APTR)/dt)*1e-2; % m/s2
    sPeriod = load('sPeriod.txt');
    [~, PSV1, ~, ~, ~, ~] = SpectrumGMs(0.05, sPeriod, a_PTR, dt);
    [~, PSV2, ~, ~, ~, ~] = SpectrumGMs(0.05, sPeriod, a_APTR, dt);
    figure
    plot(sPeriod,PSV1.*1e2)
    hold on
    plot(sPeriod,PSV2.*1e2)
    legend('PTR','APTR')
    set(gcf,'unit','centimeters','position',[10 10 30 10])
    % Results
    v_OR = v_series;        % cm/s
    v_ANR = v_APTR + v_BGR; % cm/s
    figure
    plot((1:length(v_OR)).*dt,v_OR)
    hold on
    plot((1:length(v_ANR)).*dt,v_ANR)
    legend('OR','ANR')
    set(gcf,'unit','centimeters','position',[10 10 30 10])
    figure
    plot((1:length(v_PTR)).*dt,v_PTR)
    hold on
    plot((1:length(v_APTR)).*dt,v_APTR)
    legend('PTR','APTR')
    set(gcf,'unit','centimeters','position',[10 10 30 10])
## FourthButterworth.m
    %% 4th-order Butterworth design by H.P.Duan; hpduan2000@csu.edu.cn
    function [v_PTR, v_BGR] = FourthButterworth(v_series, dt, Tp, alpha_1)
        fc = 1/(alpha_1*Tp-dt);       % truncation frequency
        fs = 1/dt;
        wn = 2*fc/fs;
        [b,a] = butter(4,wn,'low');   % design 4th-order butterworth
        v_PTR = filter(b,a,v_series); % pulse-type record-PTR
        v_BGR = v_series - v_PTR;     % high-frequency background record-BGR
    end
## getAmpDtPEER.m
    %% H.P.Duan; hpduan2000@csu.edu.cn
    function [wave, dt, NPTS, rsn] = getAmpDtPEER(filePath,fileName)
        fileid_shock = fopen([filePath,'/',fileName]);
        rsn = sscanf(char(fileName), 'RSN %f _');
        for i = 1:4
            timeline = fgetl(fileid_shock);
        end
        [time_infor,~] = strsplit(timeline,{'=',' ',','},...
            'CollapseDelimiters',true); % split the time information line
        NPTS = str2double(time_infor{2}); % read NPTS as num
        dt = str2double(time_infor{4}); % read DT as num
        Ce = textscan(fileid_shock,'%f');
        wave = zeros(NPTS,1);
        for i = 1:size(Ce{1},1)
            for j = 1:size(Ce,2)
                num = (i-1) * size(Ce,2) + j;
                wave(num) = Ce{j}(i);
            end
        end
        % Delet NaN
        wave(isnan(wave)) = [];
        fclose(fileid_shock);
    end
## getFolderList.m
    %% H.P.Duan; hpduan2000@csu.edu.cn
    function list = getFolderList(path)
        file = dir(path);
        list = {file.name}';
        list = list(3:end);
    end
## SpectrumGMs.m
    %% Spectrum of GMs by H.P.Duan; hpduan2000@csu.edu.cn
    function [PSA, PSV, SD, SA, SV, OUT] = SpectrumGMs(xi, sPeriod, gacc, dt)
    % Input:
    %       xi = ratio of critical damping (e.g., 0.05)
    %  sPeriod = vector of spectral periods
    %     gacc = input acceleration time series in cm/s2
    %       dt = sampling interval in seconds (e.g., 0.005)
    % Output:
    %      PSA = Pseudo-spectral acceleration ordinates
    %      PSV = Pseudo-spectral velocity ordinates
    %       SD = Spectral displacement ordinates
    %       SA = Spectral acceleration ordinates
    %       SV = Spectral velocity ordinates
    %      OUT = Time series of acceleration, velocity and displacemet response of SDF
    % Ref:
    % Wang, L.J. (1996). Processing of near-field earthquake accelerograms:
    % Pasadena, California Institute of Technology.

    vel = cumtrapz(gacc)*dt;
    disp = cumtrapz(vel)*dt;

    % Spectral solution
    for i = 1:length(sPeriod)
        omegan = 2*pi/sPeriod(i);
        C = 2*xi*omegan;
        K = omegan^2;
        y(:,1) = [0;0];
        A = [0 1; -K -C]; Ae = expm(A*dt); AeB = A\(Ae-eye(2))*[0;1];
        
        for k = 2:numel(gacc)
            y(:,k) = Ae*y(:,k-1) + AeB*gacc(k);
        end
        
        displ = (y(1,:))';                          % Relative displacement vector (cm)
        veloc = (y(2,:))';                          % Relative velocity (cm/s)
        foverm = omegan^2*displ;                    % Lateral resisting force over mass (cm/s2)
        absacc = -2*xi*omegan*veloc-foverm;         % Absolute acceleration from equilibrium (cm/s2)
        
        % Extract peak values
        displ_max(i) = max(abs(displ));             % Spectral relative displacement (cm)
        veloc_max(i) = max(abs(veloc));             % Spectral relative velocity (cm/s)
        absacc_max(i) = max(abs(absacc));           % Spectral absolute acceleration (cm/s2)
        
        foverm_max(i) = max(abs(foverm));           % Spectral value of lateral resisting force over mass (cm/s2)
        pseudo_acc_max(i) = displ_max(i)*omegan^2;  % Pseudo spectral acceleration (cm/s)
        pseudo_veloc_max(i) = displ_max(i)*omegan;  % Pseudo spectral velocity (cm/s)
        
        PSA(i) = pseudo_acc_max(i);                 % PSA (cm/s2)
        SA(i)  = absacc_max(i);                     % SA (cm/s2)
        PSV(i) = pseudo_veloc_max(i);               % PSV (cm/s)
        SV(i)  = veloc_max(i);                      % SV (cm/s)
        SD(i)  = displ_max(i);                      % SD  (cm)
    
        % Time series of acceleration, velocity and displacement response of
        % SDF oscillator
        OUT.acc(:,i) = absacc;
        OUT.vel(:,i) = veloc;
        OUT.disp(:,i) = displ;
    end
## SynthesisPulse.m
    %% Artificial synthesis pulse GMs by H.P.Duan; hpduan2000@csu.edu.cn
    function v_APTR = SynthesisPulse(v_series, dt, Ap, fp, t0, gama, v_)
        % Random vibration simulation
        % .....Start: set key pars
        v = v_*(pi/180);
        % .....Must ensure the gama > 1;
        temp = 1;
        for t = dt:dt:dt*length(v_series)
            temp = temp; %#ok
            if (t0-gama/(2*fp))<=t && t<=(t0+gama/(2*fp))
                v_APTR(temp,:) = Ap*(1/2)*(1+cos(2*pi*fp*(t-t0)/gama))*cos(2*pi*fp*(t-t0)+v); %#ok
            else
                v_APTR(temp,:) = 0; %#ok
            end
            temp = temp + 1;
        end
        % .....End synthesis
    end





       